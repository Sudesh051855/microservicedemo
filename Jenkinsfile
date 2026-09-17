pipeline {

    agent any

    environment {

        AWS_REGION = 'us-east-1'

        ECR_REGISTRY = '550822831139.dkr.ecr.us-east-1.amazonaws.com'

        NEXUS_REGISTRY = 'speshway-live-dev-alb-518584824.us-east-1.elb.amazonaws.com:8082'

    }

    stages {

        stage('Checkout') {

            steps {

                checkout scm

            }

        }

        stage('Maven Build') {

            steps {

                sh 'mvn clean package -DskipTests'

            }

        }

        stage('SonarQube Analysis') {

            steps {

                withSonarQubeEnv('SonarQube') {

                    sh 'mvn sonar:sonar'

                }

            }

        }

        stage('Docker Build') {

            steps {

                sh '''

                    for service in auth-service user-service audit-service contact-service customer-service file-service gateway-service invoice-service

                    do

                        echo "Building $service"

                        docker build -t $service:latest ./$service

                    done

                '''

            }

        }

        stage('Push 2 Services to ECR') {

            steps {

                withCredentials([[

                    $class: 'AmazonWebServicesCredentialsBinding',

                    credentialsId: 'aws-credentials'

                ]]) {

                    sh '''

                        aws ecr get-login-password --region $AWS_REGION | \

                        docker login --username AWS --password-stdin $ECR_REGISTRY

                        docker tag auth-service:latest $ECR_REGISTRY/auth-service:latest

                        docker push $ECR_REGISTRY/auth-service:latest

                        docker tag user-service:latest $ECR_REGISTRY/user-service:latest

                        docker push $ECR_REGISTRY/user-service:latest

                    '''

                }

            }

        }

        stage('Push 6 Services to Nexus') {

            steps {

                withCredentials([usernamePassword(

                    credentialsId: 'nexus-creds',

                    usernameVariable: 'NEXUS_USER',

                    passwordVariable: 'NEXUS_PASS'

                )]) {

                    sh '''

                        echo "$NEXUS_PASS" | docker login $NEXUS_REGISTRY \

                        -u "$NEXUS_USER" --password-stdin

                        for service in audit-service contact-service customer-service file-service gateway-service invoice-service

                        do

                            echo "Pushing $service to Nexus"

                            docker tag $service:latest $NEXUS_REGISTRY/$service:latest

                            docker push $NEXUS_REGISTRY/$service:latest

                        done

                    '''

                }

            }

        }

    }

    post {

        success {

            echo 'CI/CD Pipeline completed successfully.'

        }

        failure {

            echo 'CI/CD Pipeline failed. Check the failed stage.'

        }

    }

}
 
