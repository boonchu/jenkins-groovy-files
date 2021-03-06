// Jenkins Groovy script
@Library('jenkins-groovy-lib@feature/using-folders') _


// Uses Declarative syntax to run commands inside a container.
pipeline {

    // Jenkins parameters
    // https://devopscube.com/declarative-pipeline-parameters/
    parameters {
        string(name: 'GIT_BRANCH_NAME', defaultValue: 'develop', description: 'Git branch to use for the build.')
        string(name: 'GIT_URL', defaultValue: 'https://github.com/boonchu/java-hello-world-with-maven.git', description: 'Git project.')
        booleanParam(name: 'DEPLOY_MODE', defaultValue: true, description: 'Toggle on/off deployment')
    }
    // * end of params *

    // environment values
    // https://e.printstacktrace.blog/jenkins-pipeline-environment-variables-the-definitive-guide/
    // https://www.lambdatest.com/blog/set-jenkins-pipeline-environment-variables-list/
    environment {
        // Maven settings.xml
        def CONFIG_FILE_UUID   = '8ac4e324-359d-4b24-9cc3-04893a7d56ce'

        def GIT_BRANCH_NAME = "${params.GIT_BRANCH_NAME}"
        def DEPLOY_MODE = "${params.DEPLOY_MODE}"

        // Sonar Settings
        def SONAR_SERVER_URL     = 'http://sonar.infra.loc'
        def SONAR_SCANNER_HASH   = '81ac0554454482e834b04c84e3d2525927f4718f'
        def SONAR_PLUGIN_VERSION = '3.9.0.2155'
        def SONAR_PROJECT_KEY    = 'java'

        // Nexus settings
        NEXUS_VERSION = "nexus3"
        // This can be http or https
        NEXUS_PROTOCOL = "http"
        // Where your Nexus is running
        NEXUS_URL = "nexus.infra.loc:8081"
        // Repository where we will upload the artifact
        NEXUS_REPOSITORY = "springboot"
        // Jenkins credential id to authenticate to Nexus OSS
        NEXUS_CREDENTIAL_ID = "nexus"

    }
    // * end of envs *

    // k8s jenkins jnlp agents
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
  - name: docker
    image:  boonchu/docker:dind
    securityContext:
      privileged: true
    env:
      - name: DOCKER_TLS_CERTDIR
        value: ""
      - name: DOCKER_DRIVER
        value: "overlay2"
'''
        }
    }
	// * end of k8s *

    // stages
    stages {
        stage('Git CheckOut and Discovery') {
            steps {
				outputs 'Git CheckOut'
				git branch: "${params.GIT_BRANCH_NAME}", url: "${params.GIT_URL}"
                script {
                    // artifact info discovery
                    def pom = read_artifact_info()
                    // https://maven.apache.org/pom.html
                    // https://stackoverflow.com/questions/53541489/updating-environment-global-variable-in-jenkins-pipeline-from-the-stage-level/53541813
                    // video - https://www.youtube.com/watch?v=KwQDxwZRZiE
                    env.ARTIFACT_PKG_VERSION="${pom.version}"
                    env.ARTIFACT_PKG_NAME="${pom.artifactId}"
                    env.ARTIFACT_PKG_SUFFIX="${pom.packaging}"
                    env.ARTIFACT_PKG_GROUP="${pom.groupId}"
                    echo "LOG->INFO : ARTIFACT_PKG_VERSION is ${env.ARTIFACT_PKG_VERSION}"
                    echo "LOG->INFO : ARTIFACT_PKG_NAME is ${env.ARTIFACT_PKG_NAME}"
                    echo "LOG->INFO : ARTIFACT_PKG_SUFFIX is ${env.ARTIFACT_PKG_SUFFIX}"
                    echo "LOG->INFO : ARTIFACT_PKG_GROUP is ${env.ARTIFACT_PKG_GROUP}"

                    folders()
                }
            }
        }

        stage('Sending Notification when started') {
            steps {
                outputs 'Sending Notification when started'
                notify 'STARTED'
            }
        }

        stage('Static Analysis and simple UnitTest') {
            steps {
				outputs 'Static Analysis and simple UnitTest'
                echo "LOG-->INFO-->SONAR_PLUGIN_VERSION : ${SONAR_PLUGIN_VERSION}"
                echo "LOG-->INFO-->SONAR_SERVER_URL : ${SONAR_SERVER_URL}"
                echo "LOG-->INFO-->SONAR_SCANNER_HASH : ${SONAR_SCANNER_HASH}"

                // maven unit test
                configFileProvider([configFile(fileId: "${CONFIG_FILE_UUID}", variable: 'MAVEN_GLOBAL_SETTINGS')]) {
                	sh """
                       mvn clean compile org.sonarsource.scanner.maven:sonar-maven-plugin:${SONAR_PLUGIN_VERSION}:sonar -f pom.xml \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.host.url=${SONAR_SERVER_URL} \
                            -Dsonar.login=${SONAR_SCANNER_HASH} \
                            --batch-mode -gs ${MAVEN_GLOBAL_SETTINGS}
                       mvn clean test -f pom.xml --batch-mode -gs $MAVEN_GLOBAL_SETTINGS
                    """
                }
            }
        }

        stage('Java Code Coverage - JaCoCo') {
            steps {
               configFileProvider([configFile(fileId: "${CONFIG_FILE_UUID}", variable: 'MAVEN_GLOBAL_SETTINGS')]) {
                   withSonarQubeEnv(installationName: 'sonarqube-server') {
                        sh """
                            mvn org.jacoco:jacoco-maven-plugin:prepare-agent -f pom.xml clean test \
                                 -Dautoconfig.skip=true -Dmaven.test.skip=false \
                                 -Dmaven.test.failure.ignore=true sonar:sonar \
                                 -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                 -Dsonar.host.url=${SONAR_SERVER_URL} \
                                 -Dsonar.login=${SONAR_SCANNER_HASH} \
                                 --batch-mode -gs ${MAVEN_GLOBAL_SETTINGS}
                           """
                    }
               }
            }
        }

        stage("Quality Gate"){
            steps{
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate(abortPipeline: true)
                        if (qg.status != 'OK') {
                           error "Pipeline aborted due to a quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Archive jar files') {
            steps {
                outputs "Archive jar files"
                // maven package
                configFileProvider([configFile(fileId: "${CONFIG_FILE_UUID}", variable: 'MAVEN_GLOBAL_SETTINGS')]) {
                	sh """
                       mvn package -DskipTests=true -f pom.xml --batch-mode -gs $MAVEN_GLOBAL_SETTINGS
                    """
                }
            }
		}

        stage('Publish jar to Nexus') {
            steps {
                // https://dzone.com/articles/jenkins-publish-maven-artifacts-to-nexus-oss-using
                // read artifact version
                script {
            		// Nexus artifact repository
                    filesByGlob = findFiles(glob: "target/*.${env.ARTIFACT_PKG_SUFFIX}");
                    echo "LOG->INFO : DEBUG ARTIFACT ${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {                
                         echo "LOG->INFO : File: ${artifactPath}, group: ${env.ARTIFACT_PKG_GROUP}, packaging: ${env.ARTIFACT_PKG_SUFFIX}, version ${env.ARTIFACT_PKG_VERSION}";
                         nexusArtifactUploader(
                            nexusVersion: env.NEXUS_VERSION,
                            protocol: env.NEXUS_PROTOCOL,
                            nexusUrl: env.NEXUS_URL,
                            groupId: env.ARTIFACT_PKG_GROUP,
                            version: env.ARTIFACT_PKG_VERSION,
                            repository: env.NEXUS_REPOSITORY,
                            credentialsId: env.NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                // Artifact generated such as .jar, .ear and .war files.
                                [artifactId: env.ARTIFACT_PKG_NAME,
                                classifier: '',
                                file: artifactPath,
                                type: env.ARTIFACT_PKG_SUFFIX],
            
                                // Lets upload the pom.xml file for additional information for Transitive dependencies
                                [artifactId: env.ARTIFACT_PKG_SUFFIX,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                         );
                    } else {
                        error "LOG->ERROR :  File: ${artifactPath}, could not be found";
                    }
                }
            
                echo "LOG->INFO : GIT_BRANCH_NAME is ${params.GIT_BRANCH_NAME}"
                echo "LOG->INFO : DEPLOY_MODE is ${params.DEPLOY_MODE}"
            }
        }

        // https://michakutz.medium.com/conditionals-in-a-declarative-pipeline-jenkinsfile-d1a4a44e93bb
        // https://stackoverflow.com/questions/53764075/declarative-pipeline-use-of-when-condition-how-to-do-nested-conditions-anyof
        // https://technology.first8.nl/how-to-start-with-declarative-jenkins-pipelines/
        // conditional expression
        stage('Building Image') {
           when {
             	anyOf {
             	    expression { params.GIT_BRANCH_NAME =~ /(develop|master)/ }
             	}
             	expression{params.DEPLOY_MODE == true}
           }
           steps {
               outputs 'Building Image'
               container('docker') {
           		withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                  		sh """
           				curl -u ${USERNAME}:${PASSWORD} http://${NEXUS_URL}/repository/springboot/info/maigo/lab/hello/maigolab_hello/1.0.2/maigolab_hello-1.0.2.jar -vvv -o target/maigolab_hello-1.0.2.jar
                  		"""
           		}
           
           		withCredentials([usernamePassword(credentialsId: 'docker-login', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
           			sh """
           				docker login -u ${USERNAME} -p ${PASSWORD}
                         	docker build -t boonchu/maigolab_hello .
                         	docker tag boonchu/maigolab_hello boonchu/maigolab_hello:${currentBuild.displayName}
           			"""
           		}
               }
           } 
        }

        stage('Deploy Image') {
           when {
               anyOf {
                   expression { params.GIT_BRANCH_NAME =~ /(develop|master)/ }
               }
               expression{params.DEPLOY_MODE == true}
           }
           steps {
               outputs 'Deploying Image'
               container('docker') {
                   withCredentials([usernamePassword(credentialsId: 'docker-login', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                   		sh """
                   			docker push boonchu/maigolab_hello:${currentBuild.displayName}
                   		"""
                   }
               }
           } 
        }
    }
	// * end of stages *

    post{
        always{
            echo "========always========"
            notify "${currentBuild.currentResult}"
        }
	}
}
