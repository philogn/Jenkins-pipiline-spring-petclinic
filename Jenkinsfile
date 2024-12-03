pipeline{
    agent any
    tools{
        jdk "jdk17"
        maven "maven3"
    }

    environment{
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "localhost:8081"
        NEXUS_REPOSITORY = "Spring"
        NEXUS_CREDENTIALS_ID = "NexusCre"
        ARTIFACT_VERSION = "${BUILD_NUMBER}"
    }
    
    stages{

        stage("Compile") {
            steps{
                sh "mvn clean package"
            }
        }

        stage("Unit Test") {
            steps{
                sh "./mvnw test"
                //Execute the test
            }

            post{
                always{
                    junit '**/target/surefire-reports/TEST-**.xml'
                    //Display reports
                }
            }
        }

        stage("Scan SonarQube"){
            steps{
                sh ''' mvn sonar:sonar -Dsonar.url=https://http://localhost:9000 -Dsonar.login=squ_12d7c8f50a9c9a188b980c224f1117edd80373da \
                -Dsonar.projectName=Spring \
                -Dsonar.projectKey=Spring \
                -Dsonar.java.binaries=. '''
            }
        }

        stage("Upload Artifact to Nexus") {
            steps {
                script {
                    // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
                    pom = readMavenPom file: "pom.xml";
                    // Find built artifact under target folder
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    // Print some info from the artifact found
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    // Extract the path from the File found
                    artifactPath = filesByGlob[0].path;
                    // Assign to a boolean response verifying If the artifact name exists
                    artifactExists = fileExists artifactPath;

                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";

                        nexusArtifactUploader(
                            nexusVersion: "$NEXUS_VERSION",
                            protocol: "$NEXUS_PROTOCOL",
                            nexusUrl: "$NEXUS_URL",
                            groupId: pom.groupId,
                            version: "$ARTIFACT_VERSION",
                            repository: "$NEXUS_REPOSITORY",
                            credentialsId: "$NEXUS_CREDENTIALS_ID",
                            artifacts: [
                                // Artifact generated such as .jar, .ear and .war files.
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging]
                            ]
                        );

                    } else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }

        stage("Deploy") {
            steps{
                sh ''' 
                rm spring*
                wget --user=$NEXUS_CREDENTIALS_ID_USR --password=$NEXUS_CREDENTIALS_ID_PSW "http://localhost:8081/repository/Spring/org/springframework/samples/spring-petclinic/36/spring-petclinic-36.jar"  '''

                script{
                    withEnv(["JENKINS_NODE_COOKIE=dontKillMe"]) {
                        sh "nohup java -jar -Dserver.port=8888 spring* &"
                    }
                }
            }
        }
    }
}