@Library('lokisharedlibs') _
pipeline {

    agent any

    

    tools {

        maven 'maven3.9.0'
  
    }

    stages{

        stage('Checkout the code') {

            steps {

            git branch: 'main',
            credentialsId: 'github_auth',
            url: 'https://github.com/icicibank-org-nov/mrdevops_java_app.git'
            sendSlackNotifications('STARTED')

            }
        }

        stage('UNIT TEST') {

            steps{

                sh "mvn test"
            }
        }

       stage('INTEGRATE maven test') {

            steps{

                sh "mvn verify -DskipUnitTests"
            }
        }

       stage('Build the package') {

            steps{

                sh "mvn clean install"
            }
        }

        stage('Build & sonarqube analysis') {

            steps{
                
                withSonarQubeEnv(credentialsId: 'sonar_auth',installationName: 'sonarqube') {
                    
                 sh "mvn clean package sonar:sonar"
                }
               
            }
        }

        stage('sonar Quality analysis') {

            steps{

                waitForQualityGate abortPipeline: false, credentialsId: 'sonar_auth'
            }
        }

        /*stage('Upload the artifact into nexus') {

            steps{
                      
                script{

                    def mavenpomVersion = readMavenPom file: 'pom.xml'

                    def nexusRepo = mavenpomVersion.endsWith("SNAPSHOT") ? "loginapp_snapshot" : "loginapp_release"
                
                sh """
                      nexusArtifactUploader artifacts: 
                      [
                        [
                            artifactId: 'spring-boot-starter-parent',
                            classifier: '',
                            file: 'target/login-release.war',
                            type: 'jar'
                        ]
                    ],
                    credentialsId: 'Nexus_crd',
                    groupId: 'com.dpt3',
                    nexusUrl: '3.110.154.84:8081',
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    repository: nexusRepo ,
                    version: '${mavenpomVersion.version}'

                """                  
                }
            }
        }*/

        stage('Build the docker image') {

            steps{

                script{

                    sh """
                      docker image build -t $JOB_NAME:v1.$BUILD_ID .
                      docker image tag $JOB_NAME:v1.$BUILD_ID lokeshsdockerhub/$JOB_NAME:v1.$BUILD_ID
                      docker image tag $JOB_NAME:v1.$BUILD_ID lokeshsdockerhub/$JOB_NAME:latest
                    """
                }
            }
        }

        stage('Push the image into dockerhub') {

            steps{

                script{

                    
                    withCredentials([string(credentialsId: 'dockerhub_auth', variable: 'dockerhub_pwd')]) {
                    sh "docker login -u lokeshsdockerhub --password ${dockerhub_pwd}"
                    sh "docker push lokeshsdockerhub/$JOB_NAME:v1.$BUILD_ID"
                    sh "docker push lokeshsdockerhub/$JOB_NAME:latest"
                    }
                  
                    
                }
            }
        }

       stage('Update the image tag in helm repo'){

        steps{
            script{
        
            """
            stage('Update Deployment File') {
        environment {
            GIT_REPO_NAME = "icicibank-org-nov"
            GIT_USER_NAME = "lokeshgithub"
        }
        steps {
            withCredentials([string(credentialsId: 'Jenkins_Github_PWD', variable: 'GITHUB_TOKEN')]) {
                sh """
                    git config user.email "lokeshreddy05690@gmail.com"
                    git config user.name "lokeshgithub"
                    BUILD_NUMBER=${BUILD_NUMBER}
                    sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" minikube-sample/deployment.yml
                    git add minikube-sample/deployment.yml
                    git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                    git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                """
            }
        }
    }

         
            """
            }

        }
       } 

    }

    post {
        always {
            emailext body: 'A Test EMail', recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']], subject: '$BUILD_NUMBER:currentBuild.result'
        }
        success {
            sendSlackNotifications(currentBuild.result)
        }
        failure {
            sendSlackNotifications(currentBuild.result)
        }
    }

}