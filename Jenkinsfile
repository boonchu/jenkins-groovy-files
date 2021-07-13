@Library('jenkins-groovy-lib@feature/using-notify') _


// Uses Declarative syntax to run commands inside a container.
pipeline {

    environment {
        // Maven settings.xml
        def CONFIG_FILE_UUID   = '8ac4e324-359d-4b24-9cc3-04893a7d56ce'

        // Sonar Settings
        def SONAR_SERVER_URL     = 'http://172.30.30.102:9000'
        def SONAR_SCANNER_HASH   = '6d401f63ef2d3cbae6c1536064077d2178bb6d2e'
        def SONAR_PLUGIN_VERSION = '3.9.0.2155'

        // Nexus settings
        NEXUS_VERSION = "nexus3"
        // This can be http or https
        NEXUS_PROTOCOL = "http"
        // Where your Nexus is running
        NEXUS_URL = "172.30.30.102:8081"
        // Repository where we will upload the artifact
        NEXUS_REPOSITORY = "springboot"
        // Jenkins credential id to authenticate to Nexus OSS
        NEXUS_CREDENTIAL_ID = "nexus"
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
    stages {

        stage('Git CheckOut') {
            steps {
				hello 'Git CheckOut'
				git branch: "develop", url: "https://github.com/boonchu/java-hello-world-with-maven.git"
				read_artifact_info
            }
        }

        stage('Sending Notification when started') {
            steps {
				hello 'Sending Notification when started'
				notify 'STARTED'
            }
        }

        stage('Static Analysis and simple UnitTest') {
            steps {
				hello 'Static Analysis and simple UnitTest'
                echo "LOG-->INFO-->SONAR_PLUGIN_VERSION : ${SONAR_PLUGIN_VERSION}"
                echo "LOG-->INFO-->SONAR_SERVER_URL : ${SONAR_SERVER_URL}"
                echo "LOG-->INFO-->SONAR_SCANNER_HASH : ${SONAR_SCANNER_HASH}"

                // maven unit test
                configFileProvider([configFile(fileId: "${CONFIG_FILE_UUID}", variable: 'MAVEN_GLOBAL_SETTINGS')]) {
                	sh """
                       mvn clean compile org.sonarsource.scanner.maven:sonar-maven-plugin:${SONAR_PLUGIN_VERSION}:sonar -f pom.xml \
                            -Dsonar.projectKey=maven-code-analysis \
                            -Dsonar.host.url=${SONAR_SERVER_URL} \
                            -Dsonar.login=${SONAR_SCANNER_HASH} \
                            -gs ${MAVEN_GLOBAL_SETTINGS}
                       mvn clean test -f pom.xml -gs $MAVEN_GLOBAL_SETTINGS
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
                                 -Dsonar.host.url=${SONAR_SERVER_URL} \
                                 -Dsonar.login=${SONAR_SCANNER_HASH} \
                                 -gs ${MAVEN_GLOBAL_SETTINGS}
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
				hello "Archive jar files"

                // maven package
                configFileProvider([configFile(fileId: "${CONFIG_FILE_UUID}", variable: 'MAVEN_GLOBAL_SETTINGS')]) {
                	sh """
                       mvn package -DskipTests=true -f pom.xml -gs $MAVEN_GLOBAL_SETTINGS
                    """
                }

                // https://dzone.com/articles/jenkins-publish-maven-artifacts-to-nexus-oss-using
                // read artifact version
                script {

                    def pom = readMavenPom file: 'pom.xml'
                    ARTIFACT_VERSION = pom.version
                    ARTIFACT_PKG_NAME = pom.packaging
                    echo "LOG->INFO : ARTIFACT_VERSION is ${ARTIFACT_VERSION}"
                    echo "LOG->INFO : ARTIFACT_PKG_NAME is ${ARTIFACT_PKG_NAME}"

                    filesByGlob = findFiles(glob: "target/*.${ARTIFACT_PKG_NAME}");
                    echo "LOG->INFO : DEBUG ARTIFACT ${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {                
                         echo "LOG->INFO : File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                         nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                // Artifact generated such as .jar, .ear and .war files.
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],

                                // Lets upload the pom.xml file for additional information for Transitive dependencies
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                         );
                    } else {
                        error "LOG->ERROR :  File: ${artifactPath}, could not be found";
                    }
                }
			}
        }

        stage('Building Image') {
            steps {
                hello "Building Image"
				script {
                	def BUILD_NAME = "${currentBuild.displayName}"
				}
                container("docker") {
					withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                   		sh """
							curl -u ${USERNAME}:${PASSWORD} http://${NEXUS_URL}/repository/springboot/info/maigo/lab/hello/maigolab_hello/1.0.2/maigolab_hello-1.0.2.jar -vvv -o target/maigolab_hello-1.0.2.jar
                          	docker build -t boonchu/maigolab_hello .
                          	docker tag boonchu/maigolab_hello boonchu/maigolab_hello:${BUILD_NAME}
                   		"""
					}
                }
            } 
        }

        stage('Deploy Image') {
            steps {
                hello "Deploying Image"
				script {
                	def BUILD_NAME = "${currentBuild.displayName}"
				}
                container("docker") {
					withCredentials([usernamePassword(credentialsId: 'docker-login', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
						sh """
							docker login -u ${USERNAME} -p ${PASSWORD}
							docker push boonchu/maigolab_hello:${BUILD_NAME}
						"""
					}
				}
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
