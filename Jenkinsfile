@Library('jenkins-groovy-lib@feature/using-notify') _

// Uses Declarative syntax to run commands inside a container.
pipeline {
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
        stage('Testing') {
			steps {
				hello "Testing"
				// default branch is master
				gitchkout("master", "https://github.com/boonchu/java-hello-world-with-maven.git")
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
