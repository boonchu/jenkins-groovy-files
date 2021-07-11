@Library('jenkins-groovy-lib@feature/using-notify') _


// Uses Declarative syntax to run commands inside a container.
pipeline {

    environment {
        // Maven settings.xml
        def CONFIG_FILE_UUID   = '8ac4e324-359d-4b24-9cc3-04893a7d56ce'

        // Sonar Settings
        def SONAR_SERVER_URL   = 'http://172.30.30.102:9000'
        def SONAR_SCANNER_HASH = '6d401f63ef2d3cbae6c1536064077d2178bb6d2e'

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
'''
        }
    }
    stages {

        stage('Git CheckOut') {
            steps {
				hello 'Git CheckOut'
				// default branch is master
				gitchkout("master", "https://github.com/boonchu/java-hello-world-with-maven.git")
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
                       mvn clean test -f pom.xml -gs $MAVEN_GLOBAL_SETTINGS
                       mvn clean compile org.sonarsource.scanner.maven:sonar-maven-plugin:${SONAR_PLUGIN_VERSION}:sonar -f pom.xml \
                            -Dsonar.projectKey=maven-code-analysis \
                            -Dsonar.host.url=${SONAR_SERVER_URL} \
                            -Dsonar.login=${SONAR_SCANNER_HASH} \
                            -gs ${MAVEN_GLOBAL_SETTINGS}
                    """
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
