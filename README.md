# Project 14: Experience Continuous Integration with Jenkins | Ansible | Artifactory | SonarQube | PHP

## Step 1: Simulating a typical CI/CD Pipeline for a PHP Based application

# Step 1.1: Set Up
# Ansible inventory should look like this:
    ├── ci
    ├── dev
    ├── pentest
    ├── preprod
    ├── prod
    ├── sit
    └── uat
# ci inventory
    [jenkins]
    <Jenkins-Private-IP-Address>

    [nginx]
    <Nginx-Private-IP-Address>

    [sonarqube]
    <SonarQube-Private-IP-Address>

    [artifact_repository]
    <Artifact_repository-Private-IP-Address>
# dev 
    [tooling]
    <Tooling-Web-Server-Private-IP-Address>

    [todo]
    <Todo-Web-Server-Private-IP-Address>

    [nginx]
    <Nginx-Private-IP-Address>

    [db:vars]
    ansible_user=ec2-user
    ansible_python_interpreter=/usr/bin/python

    [db]
    <DB-Server-Private-IP-Address>
# pentest
    [pentest:children]
    pentest-todo
    pentest-tooling

    [pentest-todo]
    <Pentest-for-Todo-Private-IP-Address>

    [pentest-tooling]
    <Pentest-for-Tooling-Private-IP-Address>
    
  # Step 1.2: Added two more roles to my Ansible playbook
  - Respective roles were
   - Sonarqube: Sonarqube is an open-source platform developed by SonarSource for continuous inspection of code quality to perform automatic reviews with static analysis of code to detect bugs, code smells, and security vulnerabilities (source: https://en.wikipedia.org/wiki/SonarQube)
  
   - Artifactory: JFrog Artifactory is a universal DevOps solution providing end-to-end automation and management of binaries and artifacts through the application delivery process that improves productivity across your development ecosystem. (source: https://www.jfrog.com/confluence/display/JFROG/JFrog+Artifactory)
   - 
# Step 1.3: Configured Ansible for Jenkins development
- Navigated to Jenkins UI and installed Blue Ocean plugin
- Clicked Blue Ocean on the left side pane and created a pipeline.
- 
  ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/1.PNG)
 - Selected the repo ansible-config-mgt plus i had initially  created a github access token.
 -  ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/2.PNG)
 -  
 -  Created Jenkinsfile
 -  Inside the Ansible project,  I created a deploy folder
  - In the deploy folder, created a file named **Jenkinsfile** (no extension)
  - -  ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/3.PNG)
  - Added the following snippet
    ```
    pipeline {
    agent any
        stages {
            stage('Build') {
                steps {
                    script {
                        sh 'echo "Building Stage"'
                    }
                }
            }
        }
    }
    ```
  
- On Ansible project in Jenkins UI  and clicked Configure
- Scroll down to Build Configuration and for script path, enter the path to the Jenkinsfile (deploy/Jenkinsfile in our case)
-   ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/4.PNG)
- Clicked Build Now on the Pipeline
- Opened Blue Ocean again to see the build in action
-   ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/5.PNG)
- Since our pipeline is multibranch, we could build all the branches in the repo independently. To see this in action, 
  - Created a new git branch and named it features/jenkinspipeline-stages
  - Added a new build stage "Test"
    ```
    pipeline {
    agent any
      stages {
        stage('Build') {
          steps {
            script {
              sh 'echo "Building Stage"'
            }
          }
        }

        stage('Test') {
          steps {
            script {
              sh 'echo "Testing Stage"'
            }
          }
        }
        }
    }
    ```
  - To make the new branch show in Jenkins UI, click Administration to exit Blue Ocean, click the project and click Scan Repository Now from the left pane
  - Refresh the page and you should see the new branch.
  - Open Blue Ocean and you should see the new branch building (or has finished building)
  -   ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/6.PNG)
  
 - Created a new branch, added more stages into the Jenkins file to simulate below phases. (Just add an `echo` command like we have in `build` and `test` stages)
       1. Package 
       2. Deploy 
       3. Clean up
  - The final pipeline outcome was 
  -    ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/7.PNG)

### Step 1.4: Running Ansible Playbook from Jenkins
- Install Ansible on your Jenkins server
  ```
  sudo apt install ansible -y
  ```
- Install Ansible plugin in Jenkins UI
  ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/8.PNG)
  
- Create Jenkinsfile from scratch (delete all the current stages in the file)
  - Added a new stage to clone the GitHub repo
    ```
    stage('Clone repo') {
      steps {
        git(branch: 'dev', url: 'https://github.com/Ellawangari/ansible-config-mgt.git')
      }
    }
    ```
    The above step clones the repo from the dev branch.

  - Added another stage to edit /etc/ansible/ansible.cfg file
    ```
    stage('Update ansible.cfg') {
      steps {
        sh ('sudo sed -i -e "/roles_path = \\/[a-z]*\\/*/a\\roles_path = $cpath" -e "/roles_path = \\/[a-z]*\\/*/d" /etc/ansible/ansible.cfg')
      }
    }
     ```
 
 - Add the next stage, to run the playbook
 - ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/9.PNG)
 
    ```
    stage('Run playbook') {
      steps {
        ansiblePlaybook(credentialsId: 'private-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/${inventory}', playbook: 'playbooks/site.yml', tags: '${tag}')
      }
    }
     ```
    
    You could add a stage that cleans your workspace after every build (whether failed, successful or otherwise)
    
    ```
    stage('Clean up') {
      steps {
        cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
      }
    }
    ```
  
  - Commit and push your changes
  - Go to Blue Ocean and trigger a build against the branch. If everything is configured properly, you should see something like this:
     - ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/10.PNG)

  ## Step 2: CI/CD Pipeline for a TODO Application
  ### Step 2.1: Configure Artifactory
  - 
- Created an Ansible role to install Artifactory (https://github.com/Ellawangari/ansible-config-mgt/tree/master/roles/artifactory)
- Ran the role against the Artifactory server
### Step 2.2: Prepare Jenkins
- Fork the php-todo repository (https://github.com/darey-devops/php-todo.git)
- Install the following packages:
  ```
  sudo apt install -y zip libapache2-mod-php phploc php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip}
  ```
  - ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/15.PNG)
- Installed Jenkins plugins
  - Plot plugin: to display tests reports and code coverage information
  - Artifactory plugin: to easily deploy artifacts to Artifactory server
  - 
 - ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/16.PNG)

- Configure Artifactory in Jenkins UI
    ```
    http://<artifactory-server-ip>:8082/artifactory
    ```
 ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/17.PNG)
 
 - Configured login details and created a generic repository
 - 
 ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/18.PNG)
 ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/19.PNG)
  ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/20.PNG)
 
 
### Step 2.3: Integrate Artifactory repository with Jenkins
- Created a dummy Jenkinsfile in the root of the php-todo repo
- Created a database and user on the database server
  ```
  CREATE DATABASE homestead;
  CREATE USER 'homestead'@'%' IDENTIFIED BY 'sePret^i';
  GRANT ALL PRIVILEGES ON * . * TO 'homestead'@'%';
  ```
   ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/21.PNG)
   

- Updated Jenkinsfile with proper configuration
  ```
  pipeline {
    agent any

    stages {

     stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }
  
    stage('Checkout SCM') {
      steps {
            git branch: 'main', url: 'https://github.com/darey-devops/php-todo.git'
      }
    }

    stage('Prepare Dependencies') {
      steps {
             sh 'mv .env.sample .env'
             sh 'composer install'
             sh 'php artisan migrate'
             sh 'php artisan db:seed'
             sh 'php artisan key:generate'
          }
        }
      }
    }
    ```
     ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/23.PNG)
     
     
    **Blockers:** One  has to install PHP Composer manually by running the following commands:
    ```
    php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
    php -r "if (hash_file('sha384', 'composer-setup.php') === '756890a4488ce9024fc62c56153228907f1545c228516cbf63f885e036d37e9a59d27d63f46af1d4d07ee0f76181c7d3') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
    php composer-setup.php
    mv composer.phar /usr/local/bin/composer
    ```
    
     ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/24.PNG)
     
- Update Jenkinsfile to include unit tests
  ```
  stage('Execute Unit Tests') {
      steps {
             sh './vendor/bin/phpunit'
      }
  ```
  
 ![alt text](https://github.com/Ellawangari/DevOps-project14/blob/main/imgs/25.PNG)
   
   
### Step 2.4: Code Quality Analysis
Most commonly used tool for php code quality analysis is **phploc**
- Add the code analysis step in Jenkinsfile
  ```
      stage('Code Analysis') {
      steps {
            sh 'phploc app/ --log-csv build/logs/phploc.csv'

      }
    }
  ```
  **Blocker: The phploc command requires sudo privileges, so I had to edit the sh script to sudo phploc app/ --log-csv build/logs/phploc.csv**
- Plot the data using Jenkins Plot plugin
  ```
      stage('Plot Code Coverage Report') {
      steps {

            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)                          ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Directories,Files,Namespaces', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'B - Structures Containers', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'C - Average Length', yaxis: 'Average Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'D - Relative Cyclomatic Complexity', yaxis: 'Cyclomatic Complexity by Structure'      
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Classes,Abstract Classes,Concrete Classes', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'E - Types of Classes', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'F - Types of Methods', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Constants,Global Constants,Class Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'G - Types of Constants', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Test Classes,Test Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'I - Testing', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'AB - Code Structure by Logical Lines of Code', yaxis: 'Logical Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Functions,Named Functions,Anonymous Functions', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'H - Types of Functions', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Interfaces,Traits,Classes,Methods,Functions,Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'BB - Structure Objects', yaxis: 'Count'
          }
        }
  ```
  CLick on the Plots button on the left pane. If everything was configured properly, you should see something like this:
  ![](imgs/plots.png)

- Package the artifacts 
  ```
  stage ('Package Artifact') {
    steps {
            sh 'zip -qr ${WORKSPACE}/php-todo.zip ${WORKSPACE}/*'
    }
  ```

- Publish packaged artifact into Artifactory
  ```
  stage ('Deploy Artifact') {
    steps {
            script { 
                 def server = Artifactory.server 'artifactory-server'
                 def uploadSpec = """{
                    "files": [{
                       "pattern": "php-todo.zip",
                       "target": "php-todo"
                    }]
                 }"""

                 server.upload(uploadSpec) 
               }
    }
  }
  ```
  **Blocker:** For this to work, you have to create a local repository on your Artifactory with package type as Generic and Repository Key as the name of the repo (php-todo in this case)
  ![](imgs/arti-deploy.png)

- Deploy application to dev environment by launching the Ansible pipeline 
  ```
  stage ('Deploy to Dev Environment') {
    steps {
    build job: 'ansible-config-mgt/main', parameters: [[$class: 'StringParameterValue', name: 'env', value: 'dev']], propagate: false, wait: true
    }
  }
  ```
  This build job tells Jenkins to trigger a job in the ansible-config-mgt pipeline. Since the ansible-config-mgt pipeline requires some parameters, we pass them using the 'parameters' value.

## Step 3: Configure SonarQube
Software Quality - The degree to which a software component, system or process meets specified requirements based on user needs and expectations.

Software Quality Gates - Quality gates are basically acceptance criteria which are usually presented as a set of predefined quality criteria that a software development project must meet in order to proceed from one stage of its lifecycle to the next one.

### Step 3.1: Tune Linux Kernel
  ```
  sudo sysctl -w vm.max_map_count=262144
  sudo sysctl -w fs.file-max=65536
  ulimit -n 65536
  ulimit -u 4096

  ##This changes do not persist after system reboot
  ```
  To enable persistence after reboot, open the /etc/security/limits.conf file and append the following.
  ```
  sonarqube   -   nofile   65536
  sonarqube   -   nproc    4096
  ```
  **Blocker:** I noticed that running SonarQube is quite compute intensive, so I used a t2.medium instance instead of a t2.micro. And the **sudo sysctl -w vm.max_map_count=262144** does not make the vm.max_map_count change persist after boot (for some odd reason), so I had to append the command to the /**etc/bash.bashrc** file.

### Step 3.2: Update system packages and Install Java and other required packages
- Update and upgrade packages
  ```
  sudo apt-get update
  sudo apt-get upgrade
  ```
- Install wget and unzip packages
  ```
  sudo apt-get install wget unzip -y
  ```
- Install OpenJDK and Java Runtime Environment (JRE)
  ```
  sudo apt-get install openjdk-11-jdk -y
  sudo apt-get install openjdk-11-jre -y
  ```
  - To set default JDK or switch to OpenJDK, run the command,
    ```
    sudo update-alternatives --config java
    ```
  - Verify JAVA is installed by running:
    ```
    java -version
    ```
### Step 3.3: Install and Setup PostgreSQL 10 Database for SonarQube
- Add PostgreSQL to repo list
  ```
  sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
  ```
- Add repo key
  ```
  wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
  ```
- Install PostgreSQL
  ```
  sudo apt-get -y install postgresql postgresql-contrib
  ```
- Start and enable PostgreSQL Database service
  ```
  sudo systemctl start postgresql
  sudo systemctl enable postgresql
  ```
- Change the default password for postgres user (to whatever password you can easily remember)
  ```
  sudo passwd postgres
  ```
  Then switch to postgres user
  ```
  su - postgres
  ```
- Create a new user for SonarQube
  ```
  createuser sonar
  ```
- Create Sonar DB and user
  - Switch to PostgreSQL shell
    ```
    psql
    ```
  - Set up encrypted password for newly created user
    ```
    ALTER USER sonar WITH ENCRYPTED password 'sonar';
    ```
  - Create a DB for SonarQube
    ```
    CREATE DATABASE sonarqube OWNER sonar;
    ```
  - Grant required privileges
    ```
    grant all privileges on DATABASE sonarqube to sonar;
    ```
  - Exit the shell
    ```
    \q
    ```
- Switch back to initial user
  ```
  exit
  ```

### Step 3.4: Install and configure SonarQube
- Download temporary files to /tmp folder
  ```
  cd /tmp && sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.3.zip
  ```
- Unzip archive to the /opt directory and move to /opt/sonarqube
  ```
  sudo unzip sonarqube-7.9.3.zip -d /opt
  sudo mv /opt/sonarqube-7.9.3 /opt/sonarqube
  ```
- Configure SonarQube
Since Sonarqube cannot be run as root user, we have to create a **sonar** user to start the service with.
  - Create a group sonar
    ```
    sudo groupadd sonar
    ```
  - Add a **sonar** user with control over /opt/sonarqube
    ```
    sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar 
    sudo chown sonar:sonar /opt/sonarqube -R
    ```
  - Open and edit SonarQube configuration file. Find and edit the following lines
    ```
    sonar.jdbc.username=sonar
    sonar.jdbc.password=sonar
    ```
    Append this line
    ```
    sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
    ```
  - Edit sonar script and set RUN_AS_USER=sonar
  - Add sonar user to sudoers file
    - First set the user's password to something you can easily remember
      ```
      sudo passwd sonar
      ```
    - Run the following command and add the following line
      ```
      sudo visudo
      ```
      ```
      sonar ALL=(ALL) NOPASSWD: ALL
      ```
  - Switch to the sonar user, switch to script directory and start the script
    ```
    su - sonar
    cd /opt/sonarqube/bin/linux-x86-64/
    ./sonar.sh start
    ```
    Output:
    ```
    Starting SonarQube...

    Started SonarQube
    ```
  - Check SonarQube logs
    ```
    tail /opt/sonarqube/logs/sonar.log
    ```
  - Configure systemd to manage SonarQube service
    - Stop the sonar service
      ```
      cd /opt/sonarqube/bin/linux-x86-64/
      ./sonar.sh stop
      ```
    - Create systemd service file
      ```
      sudo vim /etc/systemd/system/sonar.service
      ```
      Paste in the following lines:
      ```
      [Unit]
      Description=SonarQube service
      After=syslog.target network.target

      [Service]
      Type=forking

      ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
      ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

      User=sonar
      Group=sonar
      Restart=always

      LimitNOFILE=65536
      LimitNPROC=4096

      [Install]
      WantedBy=multi-user.target
      ```
      Save and exit
    - Use systemd to manage the service
      ```
      sudo systemctl start sonar
      sudo systemctl enable sonar
      sudo systemctl status sonar
      ```
### Step 3.5: Access Sonarqube
- Access the SonarQube by going to http://\<sonar-qube-ip>/9000, use **admin** as your username and password
![](imgs/sonar.png)
- If you are having a timeout error, check the security group attached to the Sonarqube instance.

### Step 3.6: Configure SonarQube and Jenkins for Quality Gate
- Install SonarQube plugin in Jenkins UI
- Navigate to Manage Jenkins > Configure System and add a SonarQube server
  - In the name field, enter **sonarqube**
  - For server URL, enter the IP of your SonarQube instance
- Generate authentication tokens to use in the Jenkins UI
  - In SonarQube UI, navigate to User > My Account > Security > Generate tokens
  - Type in the name of the token and click Generate

- Configure SonarQube webhook for Jenkins
  -  Navigate to Administration > Configuration > Webhooks > Create
  - Enter the name
  - Enter URL as http://\<jenkins-server-ip>:8080/sonar-webhook/

- Setup SonarScanner for Jenkins
  - In Jenkins UI, go to Manage Jenkins > Global Tool Configuration
  - Look for SonarQube Scanner
  - Click 'Add SonarQube Scanner' and enter the scanner name as 'SonarQubeScanner'
  - Check the 'Install automatically' box
  - Select 'Install from Maven Central'

- Add the following build stage for Quality Gate
  ```
  stage('SonarQube Quality Gate') {
      environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
            }

        }
    }
  ```
  Save and run the pipeline to install the scanner (you might get an error initially)

- Edit sonar-scanner.properties file
  ```
  sudo vim /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/sonar-scanner.properties
  ```
  Paste in the following lines:
  ```
  sonar.host.url=http://<SonarQube-Server-IP-address>:9000
  sonar.projectKey=php-todo
  #----- Default source code encoding
  sonar.sourceEncoding=UTF-8
  sonar.php.exclusions=**/vendor/**
  sonar.php.coverage.reportPaths=build/logs/phploc.csv
  sonar.php.tests.reportPath=reports/unitreport.xml
  ```
  Install xdebug and configure the ini file, you can get the path to the file by running
  ```
  php --ini | grep xdebug
  ```
  And paste in:
  ```
  xdebug.mode=coverage
  ```
  **Blocker:** the sonar.php.coverage.reportPaths and sonar.php.tests.reportPath should point to the files in build/coverage/ or else the Quality Gate would be incomplete
- To generate pipleline code from Jenkins UI, go to Dashboard, click on the pipeline project (php-todo) and scroll down to Pipeline Syntax
- If everything was configured properly, you should see something like this: ![](imgs/endtoend.png)
- Navigate to your php-todo dashboard on SonarQube UI ![](imgs/qualitygate.png)

### Step 3.7: Conditionally Deploy to Higher Environments
- Include a **when** condition to execute the Quality Gate stage only when the running branch is develop, hotfix, release, main or master
  ```
  when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
  ```
- Add a timeout step to abort to Quality Gate stage after 1 minute (or any time you like)
  ```
      timeout(time: 1, unit: 'MINUTES') {
        waitForQualityGate abortPipeline: true
    }
  ```
  Quality Gate stage snippet should look like this:
  ```
  stage('SonarQube Quality Gate') {
      when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
            }
            timeout(time: 1, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }
  ```
  You should get the following when you run the pipeline:
  ![](imgs/timeout.png)
  For specific branches (Quality Gate does not execute I am running from 'testing' branch):
  ![](imgs/branch-testing.png)

## Step 4: Configure Jenkins slave servers
- Spin up a new EC2 Instance
  - Install Java and Jenkins
  - Install necessary packages from Steps 1.4, 2.3 Blocker and 2.4
  - Create new user to be used by jenkins
    ```
    sudo useradd -d /var/lib/jenkins jenkins
    sudo passwd jenkins
    ```
  - Create the default root directory and set permissions
    ```
    sudo mkdir /var/lib/jenkins
    sudo chown -R jenkins:jenkins /var/lib/jenkins
    ```
  - Generate SSH Key for jenkins to use when logging in
    ```
    su - jenkins
    ssh-keygen -t rsa -C "Jenkins agent key"
    ```
  - Add id_rsa.pub to authorized_keys file
    ```
    # cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    # chmod 600 ~/.ssh/authorized_keys
    ```
  - Copy SSH private key to clipboard
    ```
    cat ~/.ssh/id_rsa
- On the main Jenkins server
  - Navigate to Manage Jenkins > Manage Nodes
  - Click New Node
  - Enter name of the node and click the 'Permanent Agent' button and click the OK button
  - Fill in the remote root directory as /var/lib/jenkins
  - Set 'Host' value as the IP of the slave node
  - For Launch Method, select Launch Agents via SSH
  - Add new Jenkins SSH with username and private key credentials with username as jenkins and private key as the private key you copied from the node
  - For Host Key Verification Strategy, select Manually trusted key validation strategy
  - Click Save

- Repeat the above steps to add more servers ![](imgs/jenkins-nodes.png)

## Step 5: Configure GitHub WebHook for Automatic Build of Pushed Code


## Step 6: Deploy to all Environments


 
