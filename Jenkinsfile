pipeline {

    agent none

    environment {

        NODE_ENV="prod"
        AWS_ACCESS_KEY=""
        AWS_SECRET_ACCESS_KEY=""
        AWS_SDK_LOAD_CONFIG="0"
        BUCKET_NAME="app-digital"
        REGION="us-east-1" 
        PERMISSION=""
        ACCEPTED_FILE_FORMATS_ARRAY=""
        VERSION="1.0.0"
    }


    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
    }
    triggers {
        cron('@daily')
    }

    stages{
        stage("Build, Test and Push Docker Image") {
            agent {  
                node {
                    label 'master'
                }
            }
            stages {

                stage('Clone repository') {
                    steps {
                        script {
                            if(env.GIT_BRANCH=='origin/prod1'){
                                checkout scm
                            }
                            sh('printenv | sort')
                            echo "My branch is: ${env.GIT_BRANCH}"
                        }
                    }
                }
                stage('Build image'){       
                    steps {
                        script {
                            print "Environment will be : ${env.NODE_ENV}"
                            docker.build("digitalhouse-devops:${env.NODE_ENV}-${env.BUILD_ID}")
                        }
                    }
                }

                stage('Test image') {
                    steps {
                        script {

                            docker.image("digitalhouse-devops:${env.NODE_ENV}-${env.BUILD_ID}").withRun('-p 8030:3000') { c ->
                                sh 'docker ps'
                                sh 'sleep 10'
                                sh 'curl http://127.0.0.1:8030/api/v1/healthcheck'
                                
                            }
                    
                        }
                    }
                }

                stage('Docker push') {
                    steps {
                        echo "Push ${env.NODE_ENV}-${env.BUILD_ID} para AWS ECR"
                        script {
                            docker.withRegistry('http://690998955571.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:awskey') {
                                docker.image("digitalhouse-devops:${env.NODE_ENV}-${env.BUILD_ID}").push()
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Producao') {
            agent {  
                node {
                    label 'prod'
                }
            }

            steps { 
                script {
                    if(env.GIT_BRANCH=='origin/prod1'){

                        docker.withRegistry('https://690998955571.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:awskey') {
                            docker.image("digitalhouse-devops:${env.NODE_ENV}-${env.BUILD_ID}").pull()
                        }

                        echo 'Deploy para Producao'
                        sh "hostname"
                       //sh "docker stop app1"
                        //sh "docker rm app1"
                        //sh "docker run -d --name app1 -p 8030:3000 933273154934.dkr.ecr.us-east-1.amazonaws.com/digitalhouse-devops:latest"
                        withCredentials([[$class:'AmazonWebServicesCredentialsBinding' 
                            , credentialsId: 'prods3']]) {
                          sh "docker run -d --name app1 -p 8030:3000 -e NODE_ENV=producao -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY -e BUCKET_NAME=dh-pi-grupo-lovelace-prod 690998955571.dkr.ecr.us-east-1.amazonaws.com/digitalhouse-devops:${env.NODE_ENV}-${env.BUILD_ID}"
                        }
                        sh "docker ps"
                        sh 'sleep 10'
                        sh 'curl http://ec2-54-204-77-140.compute-1.amazonaws.com:8030/api/v1/healthcheck'

                    }
                }
            }

        }

    }
}
