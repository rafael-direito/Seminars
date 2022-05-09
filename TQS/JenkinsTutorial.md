
# Jenkins Demonstration

### Resources

For this demo, the following resources were used:

* Jenkins Instance - Running on [10.0.13.29:8080](10.0.13.29:8080) and atnog-cicd-classes.av.it.pt:9000
* SonarQube Instance - Running on [10.0.13.29:9000](10.0.13.29:9000)
* VM with testing environment - Running on [10.0.12.78:9005](10.0.12.78:9005)
* VM with staging environment - Running on atnog-cicd-classes.av.it.pt:8080
* VM with testing environment - Running on atnog-cicd-classes.av.it.pt


# 1. Base Configuration

## 1.1 Jenkins Configuration
In the Jenkins VM, install all the required resources:

``` bash
# Install Java 8 (this project was developed using java 8)
sudo apt-get install openjdk-8-jdk -y
# if you have multiple java version installed, please update your default one to java 8, since this project was developed using java 8
sudo update-alternatives --config java
# Install Maven
sudo apt install maven -y

# For the User Acceptance Tests
sudo apt install firefox -y
# Get the firefox gecko driver
wget https://github.com/mozilla/geckodriver/releases/download/v0.31.0/geckodriver-v0.31.0-linux64.tar.gz
tar -xvf geckodriver-v0.31.0-linux64.tar.gz
chmod +x geckodriver
sudo mv geckodriver /usr/local/bin
```

## 1.2 Connect your GitHub Repository to Jenkins

This step is required for you to have automated builds every time new code is submitted to your code base.

Start by creating a webwook in github to enable this.
To do so, follow this tutorial: [How to Integrate Jenkins with GitHub](https://www.cprime.com/resources/blog/how-to-integrate-jenkins-github/) 

As you can see in the image below, Jenkins is expecting that pipeline script exists in your repository. In this case, it is expecting this script to exist in the root of your repository, with the file name "Jenkinsfile". We can create a pipeline configuration file in this location or in anywhere else. You'll just need to update the highlighted field, in Jenkins.

![enter image description here](https://i.imgur.com/9fJ4rep.png)

Don't forget to create a Jenkins pipeline!


## 1.2 Configure every thing you'll need on Jenkins

``` bash
# Install Java 8 (this project was developed using java 8)
sudo apt-get install openjdk-8-jdk -y
# if you have multiple java version installed, please update your default one to java 8, since this project was developed using java 8
sudo update-alternatives --config java
# Install Maven
sudo apt install maven -y

# For the User Acceptance Tests
sudo apt install firefox -y
# Get the firefox gecko driver
wget https://github.com/mozilla/geckodriver/releases/download/v0.31.0/geckodriver-v0.31.0-linux64.tar.gz
tar -xvf geckodriver-v0.31.0-linux64.tar.gz
chmod +x geckodriver
sudo mv geckodriver /usr/local/bin
```


## 1.3 Try to run some Unit Tests

On the root of your git repository create a file name `Jenkinsfile` with the following content:

``` groovy
node{
	
	git branch: "master", url: "https://github.com/rafael-direito/WeatherApp-QA"

	stage ('Unit Tests') {
		dir('rest_api') {
			sh "mvn -Dtest=TestCache test"
			sh "mvn -Dtest=ConstantsTest test"
			sh "mvn -Dtest=TestConverters test"
			sh "mvn -Dtest=TestCalculations test"
		}
	}
}
```

Now, submit this file to your repository and check the Jenkins UI to verify if a building process was triggered. You should see something like this (but jus with one stage -> "Unit Tests") : 


![enter image description here](https://i.imgur.com/3PBeIUB.png)


# 2. Configure the Static Code Analysis Mechanisms

To do so, follow this tutorial: [# Add SonarQube quality gates to your Jenkins build pipeline](https://tomgregory.com/sonarqube-quality-gates-in-jenkins-build-pipeline/)

You might have to create a Jenkins Credential with the access token to SonarQube.

Now, we will have to add a new stage to our pipeline, in order to run the static code analysis in Sonarqube:

```groovy
...
	stage('SonarQube analysis') {
		dir('rest_api') {
			withSonarQubeEnv('Sonar') {
				sh "mvn sonar:sonar"
			}
		}
	}
  
	stage("Did the build passed the Quality Gates?") {
		waitForQualityGate abortPipeline: true
	}
...
```

The "SonarQube analysis" stage will invoke a static code analysis, and the second stage "Did the build passed the Quality Gates?" will force Jenkins to wait for the results of this analysis, in order to proceed with the build.

Commit some files to your repository, to check if the static code analysis is being invoked.

# 3. Integration and User Acceptance Tests 

Since we are working with an old project (2016) the integration and user acceptance tests have to performed with the application running. Thus, we will deploy the application to a testing environment, which will be used to perform these tests.

So, we will configure the Jenkins SSH Agent to deploy the application to the testing environment. To install and configure this plugin follow this tutorial: [Jenkins SSH Plugin Tutorial](https://plugins.jenkins.io/ssh-agent/)

Then, we will need to add the following stages to our pipeline:

``` groovy
...
     stage ('Deploy to Testing Environment') {
        dir('rest_api') {
           
            // Package the application 
            sh "mvn clean package -Dmaven.test.skip"
            def jar_file_location = sh (script: "ls target/*.jar", returnStdout: true).trim()
            def jar_file_name = jar_file_location.split('/')[1]

            sshagent(credentials : ['atnog-cicd-classes.av.it.pt-ssh']) {

                sh "scp -o StrictHostKeyChecking=no '${jar_file_location}' jenkins@10.0.12.78:~/"
                sh "ssh -o StrictHostKeyChecking=no jenkins@10.0.12.78  bash run_WeatherApp-QA_testing.sh '${jar_file_name}'"
            }
        }
    }

    stage ('Integration Tests - Internal') {
        dir('rest_api') {
           
            // Wait for the application to be ready (max timeout -> 2 min.)
            def count = 1
            def app_running = false
            while(count <= 12) {
                echo "Checking if the application is running on10.0.12.78:9005 (try: $count)"
                status = sh (script: "curl -I http://10.0.12.78:9005", returnStatus: true)
                if (status == 0) {
                    app_running = true
                    echo "Application is running on 10.0.12.78:9005"
                    break
                }
                echo "Sleeping for 10 seconds..."
                sleep(10)
                count++
            }

            // If the application is not running, fail the test
            if (!app_running) {
                echo "Application is not running on 10.0.12.78:9005"
                error("Application is not running on 10.0.12.78:9005. Cannot continue with the tests.")

            }

            // Update App Location + Run the Tests
            sh "echo 'package weather_app.restapi.mappings;public class Constants{public static final String BASE_URL = \"http://10.0.12.78:9005\";}' > src/test/java/weather_app/restapi/mappings/Constants.java"
            sh "mvn clean test -Dtest=TemperatureResourcesTest"
            sh "mvn clean test -Dtest=ForecastsResourcesTest test"
            sh "mvn clean test -Dtest=HumidityResourcesTest test"
            

            // Kill the application
            sh "kill -9 `lsof -t -i:8081` || true"
        }
    }

    stage ('User Acceptance Tests') {
        dir('rest_api') {
           
            def count = 1
            def app_running = false
            while(count <= 12) {
                echo "Checking if the application is running on 10.0.12.78:9005 (try: $count)"
                status = sh (script: "curl -I http://10.0.12.78:9005", returnStatus: true)
                if (status == 0) {
                    app_running = true
                    echo "Application is running on 10.0.12.78:9005"
                    break
                }
                echo "Sleeping for 10 seconds..."
                sleep(10)
                count++
            }

            // If the application is not running, fail the test
            if (!app_running) {
                echo "Application is not running on 10.0.12.78:9005"
                error("Application is not running on 10.0.12.78:9005. Cannot continue with the tests.")

            }
        
            // Update App Location + Run the Tests
            sh """echo 'package weather_app.web.controllers;public class Constants{public static final String BASE_URL = \"http://10.0.12.78:9005\";}' > src/test/java/weather_app/web/controllers/Constants.java"""
            sh "mvn -Dtest=GeneralForecastTest test"

        }
    }

    stage ('Stop Deployment on Testing Environment') {
        sshagent(credentials : ['atnog-cicd-classes.av.it.pt-ssh']) {
                sh "ssh -o StrictHostKeyChecking=no jenkins@10.0.12.78  bash stop_process_on_port.sh 9005"
            }
    }
...
```

As you can see in the pipeline configuration, Jenkins will invoke two bash scripts inside the 10.0.12.78 VM. These scripts are the following:

---
`run_WeatherApp-QA_testing.sh`
```bash
#!/bin/bash
kill -9 $(lsof -t -i:9005) ||  true
java -jar  -Dserver.port=9005 -Dserver.address=0.0.0.0 $1 &>/dev/null &>/dev/null &
```

---
`stop_process_on_port.sh`
```bash
#!/bin/bash
kill -9 $(lsof -t -i:$1) ||  true
```
---

As you can see, before running the tests, we update the location of the application under testing:

```groovy
// Update App Location + Run the Tests 
sh """echo 'package weather_app.web.controllers;public class Constants{public static final String BASE_URL = \"http://10.0.12.78:9005\";}' > src/test/java/weather_app/web/controllers/Constants.java""" 
sh "mvn -Dtest=GeneralForecastTest test"
```

By now, if you run your pipeline you should see the following

![enter image description here](https://i.imgur.com/OIG6bzY.png)


# 3. Staging and Production Environments

If all the tests pass, we can now deploy our application to a staging environment (continuous delivery) or/and to a production environment (continuous deployment). The approach to this is similar to the one presented in the previous section.

Add this stages to your pipeline:

```groovy
...
stage ('Deploy to Staging') {
        dir('rest_api') {
           
            // Package the application
            sh "mvn clean package -Dmaven.test.skip"
            def jar_file_location = sh (script: "ls target/*.jar", returnStdout: true).trim()
            def jar_file_name = jar_file_location.split('/')[1]

            sshagent(credentials : ['atnog-cicd-classes.av.it.pt-ssh']) {

                sh "scp -o StrictHostKeyChecking=no '${jar_file_location}' jenkins@10.0.12.78:~/"
                sh "ssh -o StrictHostKeyChecking=no jenkins@10.0.12.78  bash run_WeatherApp-QA_staging.sh '${jar_file_name}'"
            }
        }
    }

    stage('Deploy to Production?') {
        def userInput = "No"
        try {
            timeout(time:120, unit:'SECONDS') {
                userInput = input(id: 'userInput', message: 'Do you want to deploy this build to production?',
                parameters: [[$class: 'ChoiceParameterDefinition', defaultValue: 'No', 
                    description:'describing choices', name:'nameChoice', choices: "Yes (DANGEROUS!)\nNo"]
                ])
            }
        } catch(err) { // timeout reached or input Aborted
            echo "Timeout reached or input Aborted"
            echo "Won't proceed to production"
        }

        println("Do you want to deploy this build to production? > " + userInput);
        
        if (userInput == "No") {
            currentBuild.result = 'SUCCESS'
        }
        else{
            dir('rest_api') {
                // Package the application
                def jar_file_location = sh (script: "ls target/*.jar", returnStdout: true).trim()
                def jar_file_name = jar_file_location.split('/')[1]

                sshagent(credentials : ['atnog-cicd-classes.av.it.pt-ssh']) {
                    sh "ssh -o StrictHostKeyChecking=no jenkins@10.0.12.78  bash run_WeatherApp-QA_production.sh '${jar_file_name}'"
                }
            }
        }        
    }
...
```

This pipeline configuration requires the existence of two additional bash scripts:

`run_WeatherApp-QA_staging.sh`
```bash
#!/bin/bash
kill -9 $(lsof -t -i:9006) ||  true
java -jar  -Dserver.port=9006 -Dserver.address=0.0.0.0 $1 &>/dev/null &>/dev/null &
```

`run_WeatherApp-QA_production.sh`
```bash
#!/bin/bash
kill -9 $(lsof -t -i:9007) ||  true
java -jar  -Dserver.port=9007 -Dserver.address=0.0.0.0 $1 &>/dev/null &>/dev/null &
```

Besides, please not the lines:

```groovy
...
		try {
            timeout(time:120, unit:'SECONDS') {
                userInput = input(id: 'userInput', message: 'Do you want to deploy this build to production?',
                parameters: [[$class: 'ChoiceParameterDefinition', defaultValue: 'No', 
                    description:'describing choices', name:'nameChoice', choices: "Yes (DANGEROUS!)\nNo"]
                ])
            }
        } catch(err) { // timeout reached or input Aborted
            echo "Timeout reached or input Aborted"
            echo "Won't proceed to production"
        }
...
```

This code forces Jenkins to ask us if we want to deploy the current build to production. If we don't, the application will only be deployed in a staging environment.

You can now try to run some tests with your entire pipeline defined:

```groovy
node{

    git branch: "master", url: "https://github.com/rafael-direito/WeatherApp-QA" 

    
    stage('SonarQube analysis') {
        dir('rest_api') {
            withSonarQubeEnv('Sonar') {
                sh "mvn sonar:sonar"
            }
        }
    }

    stage("Did the build passed the Quality Gates?") {
            waitForQualityGate abortPipeline: true
    }

    stage ('Unit Tests') {
        dir('rest_api') {
            sh "mvn -Dtest=TestCache test"
            sh "mvn -Dtest=ConstantsTest test"
            sh "mvn -Dtest=TestConverters test"
            sh "mvn -Dtest=TestCalculations test"
        }
    }
    
    stage ('Integration Tests - External') {
        dir('rest_api') {
            sh "mvn -Dtest=IpmaCallsTest test"
        }
    }
    
    stage ('Deploy to Testing Environment') {
        dir('rest_api') {
           
            // Package the application 
            sh "mvn clean package -Dmaven.test.skip"
            def jar_file_location = sh (script: "ls target/*.jar", returnStdout: true).trim()
            def jar_file_name = jar_file_location.split('/')[1]

            sshagent(credentials : ['atnog-cicd-classes.av.it.pt-ssh']) {

                sh "scp -o StrictHostKeyChecking=no '${jar_file_location}' jenkins@10.0.12.78:~/"
                sh "ssh -o StrictHostKeyChecking=no jenkins@10.0.12.78  bash run_WeatherApp-QA_testing.sh '${jar_file_name}'"
            }
        }
    }

    stage ('Integration Tests - Internal') {
        dir('rest_api') {
           
            // Wait for the application to be ready (max timeout -> 2 min.)
            def count = 1
            def app_running = false
            while(count <= 12) {
                echo "Checking if the application is running on10.0.12.78:9005 (try: $count)"
                status = sh (script: "curl -I http://10.0.12.78:9005", returnStatus: true)
                if (status == 0) {
                    app_running = true
                    echo "Application is running on 10.0.12.78:9005"
                    break
                }
                echo "Sleeping for 10 seconds..."
                sleep(10)
                count++
            }

            // If the application is not running, fail the test
            if (!app_running) {
                echo "Application is not running on 10.0.12.78:9005"
                error("Application is not running on 10.0.12.78:9005. Cannot continue with the tests.")

            }

            // Update App Location + Run the Tests
            sh "echo 'package weather_app.restapi.mappings;public class Constants{public static final String BASE_URL = \"http://10.0.12.78:9005\";}' > src/test/java/weather_app/restapi/mappings/Constants.java"
            sh "mvn clean test -Dtest=TemperatureResourcesTest"
            sh "mvn clean test -Dtest=ForecastsResourcesTest test"
            sh "mvn clean test -Dtest=HumidityResourcesTest test"
        }
    }

    stage ('User Acceptance Tests') {
        dir('rest_api') {
           
            def count = 1
            def app_running = false
            while(count <= 12) {
                echo "Checking if the application is running on 10.0.12.78:9005 (try: $count)"
                status = sh (script: "curl -I http://10.0.12.78:9005", returnStatus: true)
                if (status == 0) {
                    app_running = true
                    echo "Application is running on 10.0.12.78:9005"
                    break
                }
                echo "Sleeping for 10 seconds..."
                sleep(10)
                count++
            }

            // If the application is not running, fail the test
            if (!app_running) {
                echo "Application is not running on 10.0.12.78:9005"
                error("Application is not running on 10.0.12.78:9005. Cannot continue with the tests.")

            }
        
            // Update App Location + Run the Tests
            sh """echo 'package weather_app.web.controllers;public class Constants{public static final String BASE_URL = \"http://10.0.12.78:9005\";}' > src/test/java/weather_app/web/controllers/Constants.java"""
            sh "mvn -Dtest=GeneralForecastTest test"

        }
    }

    stage ('Stop Deployment on Testing Environment') {
        sshagent(credentials : ['atnog-cicd-classes.av.it.pt-ssh']) {
                sh "ssh -o StrictHostKeyChecking=no jenkins@10.0.12.78  bash stop_process_on_port.sh 9005"
            }
    }
    
    stage ('Deploy to Staging') {
        dir('rest_api') {
           
            // Package the application
            sh "mvn clean package -Dmaven.test.skip"
            def jar_file_location = sh (script: "ls target/*.jar", returnStdout: true).trim()
            def jar_file_name = jar_file_location.split('/')[1]

            sshagent(credentials : ['atnog-cicd-classes.av.it.pt-ssh']) {

                sh "scp -o StrictHostKeyChecking=no '${jar_file_location}' jenkins@10.0.12.78:~/"
                sh "ssh -o StrictHostKeyChecking=no jenkins@10.0.12.78  bash run_WeatherApp-QA_staging.sh '${jar_file_name}'"
            }
        }
    }

    stage('Deploy to Production?') {
        def userInput = "No"
        try {
            timeout(time:120, unit:'SECONDS') {
                userInput = input(id: 'userInput', message: 'Do you want to deploy this build to production?',
                parameters: [[$class: 'ChoiceParameterDefinition', defaultValue: 'No', 
                    description:'describing choices', name:'nameChoice', choices: "Yes (DANGEROUS!)\nNo"]
                ])
            }
        } catch(err) { // timeout reached or input Aborted
            echo "Timeout reached or input Aborted"
            echo "Won't proceed to production"
        }

        println("Do you want to deploy this build to production? > " + userInput);
        
        if (userInput == "No") {
            currentBuild.result = 'SUCCESS'
        }
        else{
            dir('rest_api') {
                // Package the application
                def jar_file_location = sh (script: "ls target/*.jar", returnStdout: true).trim()
                def jar_file_name = jar_file_location.split('/')[1]

                sshagent(credentials : ['atnog-cicd-classes.av.it.pt-ssh']) {
                    sh "ssh -o StrictHostKeyChecking=no jenkins@10.0.12.78  bash run_WeatherApp-QA_production.sh '${jar_file_name}'"
                }
            }
        }        
    }
}

```

# Disclaimer

This is very raw tutorial. If you wish to apply this tutorial in your projects and need additional help, send me an email to [rdireito@av.it.pt](mailto:rdireito@av.it.pt), or tag me at Slack.



