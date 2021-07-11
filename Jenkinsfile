@Library('jenkins-groovy-lib@feature/using-notify') _


// Uses Declarative syntax to run commands inside a container.
pipeline {

    environment {
        def CONFIG_FILE_UUID   = '8ac4e324-359d-4b24-9cc3-04893a7d56ce'
    }

    agent {
        kubernetes {
            defaultContainer 'openjdk11'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: openjdk11
    image: maven:3.8.1-openjdk-11
    command:
    - sleep
    args:
    - infinity
'''
        }
    }
    stages {
        stage('Call Hello()') {
            steps {
                // sh 'hostname'
				hello "World!"
            }
        }
        stage('Call Notify_Build()') {
            steps {
                // sh 'hostname'
				notify 'STARTED'
            }
        }
        stage('Testing and archive jar files') {
			steps {
				hello "Testing"
				// default branch is master
				gitchkout("master", "https://github.com/boonchu/java-hello-world-with-maven.git")
                configFileProvider([configFile(fileId: "${CONFIG_FILE_UUID}", variable: 'MAVEN_GLOBAL_SETTINGS')]) {
                	sh """
                           mvn clean test -f pom.xml -gs $MAVEN_GLOBAL_SETTINGS && mvn deploy -f pom.xml -gs $MAVEN_GLOBAL_SETTINGS
                    """
                }
			}
        }
        stage('Building Image') {
            steps {
                hello "Building Image"
            } 
        }
        stage('Deploy Image') {
            steps {
                hello "Deploying Image"
            } 
        }
    }
    post{
        always{
            echo "========always========"
            notify "${currentBuild.currentResult}"
        }
	}
}
