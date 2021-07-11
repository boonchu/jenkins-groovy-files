@Library('jenkins-groovy-lib@feature/using-notify') _

// Uses Declarative syntax to run commands inside a container.
pipeline {
    agent {
        kubernetes {
            // Rather than inline YAML, in a multibranch Pipeline you could use: yamlFile 'jenkins-pod.yaml'
            // Or, to avoid YAML:
            // containerTemplate {
            //     name 'shell'
            //     image 'ubuntu'
            //     command 'sleep'
            //     args 'infinity'
            // }
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: shell
    image: ubuntu
    command:
    - sleep
    args:
    - infinity
'''
            // Can also wrap individual steps:
            // container('shell') {
            //     sh 'hostname'
            // }
            defaultContainer 'shell'
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
            notify currentBuild.result
        }
	}
}
