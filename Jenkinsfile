pipeline {

    agent any

	/*tools {
        maven "MAVEN3"
    }*/

    environment {
        registry = "nadun2005/vproappdock"
        registryCredential = 'docker-cred'
    }

    stages{

        stage('Checkout') {
            steps {
              echo 'Checking out code...'
              git branch: 'main', credentialsId: 'GithubToken', url: 'https://github.com/Nadun-Kaveesha/vprofile-app.git'
            }
        }

        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                SONAR_URL = "http://3.80.34.209:9000"
            }

            steps {
                withCredentials([string(credentialsId: 'newultimatesonar', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
          steps {
            script {
              dockerImage = docker.build registry + ":V$BUILD_NUMBER"
            }
          }
        }

        stage('Upload Image'){
          steps{
            script {
              docker.withRegistry('', registryCredential) {
                dockerImage.push("V$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }

        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:V$BUILD_NUMBER"
          }
        }



        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "vprofile-manifest"
                GIT_USER_NAME = "Nadun-Kaveesha"
            }
            steps {
                withCredentials([string(credentialsId: 'GithubToken', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config user.email "nadunkaveesha2018@gmail.com"
                        git config user.name "Nadun-Kaveesha"
                        BUILD_NUMBER=${BUILD_NUMBER}
                        sed -i "s|appimage: .*|appimage: ${registry}:V${BUILD_NUMBER}|" vprofile-manifest/helm/vprofilecharts/values.yaml
                        git add helm/vprofilecharts/values.yaml
                        git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                    '''
                }
            }
        }
    }


}
