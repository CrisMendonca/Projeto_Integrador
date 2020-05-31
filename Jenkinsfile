pipeline {

    agent none

    environment {

        NODE_ENV="homolog"
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
                            if(env.GIT_BRANCH=='origin/homolog1'){
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
                            //print "${env.BUILD_ID}"
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
                        echo "Push image version ${env.BUILD_ID} para AWS ECR"
                        script {
                            docker.withRegistry('https://690998955571.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:awskey') {
                                docker.image("digitalhouse-devops:${env.NODE_ENV}-${env.BUILD_ID}").push()
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Homolog') {
            agent {  
                node {
                    label 'homolog'
                }
            }
            steps { 
                script {
                    if(env.GIT_BRANCH=='origin/homolog1'){
 
                        docker.withRegistry('https://690998955571.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:awskey') {
                            docker.image("digitalhouse-devops:${env.NODE_ENV}-${env.BUILD_ID}").pull()
                        }

                        echo 'Deploy para Homologacao'
                        sh "hostname"
                        //sh "docker stop app1 || true && docker rm rabbitmq || app1"
                        sh "docker ps -q --filter 'name=app1' | grep -q . && docker stop app1 && docker rm -fv app1"
                       //sh "docker run -d --name app1 -p 8030:3000 690998955571.dkr.ecr.us-east-1.amazonaws.com/digitalhouse-devops:${env.BUILD_ID}"
                        withCredentials([[$class:'AmazonWebServicesCredentialsBinding' 
                            , credentialsId: 'homologs3']]) {
                        sh "docker run -d --name app1 -p 8030:3000 -e NODE_ENV=homologacao -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY -e BUCKET_NAME=dh-pi-grupo-lovelace-homolog 690998955571.dkr.ecr.us-east-1.amazonaws.com/digitalhouse-devops:${env.NODE_ENV}-${env.BUILD_ID}"
                        }

                        sh "docker ps"
                        sh 'sleep 10'
                        sh 'curl http://ec2-3-84-5-112.compute-1.amazonaws.com:8030/api/v1/healthcheck'
                    }
                }
            }

        }
    }
}
