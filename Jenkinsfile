pipeline {

agent any

environment {

    AWS_REGION = 'us-east-1'

    ECR_REGISTRY = '550822831139.dkr.ecr.us-east-1.amazonaws.com'

    NEXUS_REGISTRY = 'speshway-live-dev-alb-518584824.us-east-1.elb.amazonaws.com:8082'

}

stages {

    stage('Test') {

        steps {

            sh 'mvn clean test'

        }

    }

    stage('SonarQube Analysis') {

        steps {

            withSonarQubeEnv('SonarQube') {

                sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar'

            }

        }

    }

    stage('Quality Gate') {

        steps {

            timeout(time: 5, unit: 'MINUTES') {

                waitForQualityGate abortPipeline: true

            }

        }

    }

    stage('Docker Build') {

        steps {

            sh '''

                services="auth-service user-service audit-service contact-service customer-service file-service gateway-service invoice-service"

                for service in $services

                do

                    echo "Building $service"

                    docker build -t $service:latest ./$service

                done

            '''

        }

    }

    stage('Trivy Image Scan') {

        steps {

            sh '''

                services="auth-service user-service audit-service contact-service customer-service file-service gateway-service invoice-service"

                for service in $services

                do

                    echo "Scanning $service"

                    trivy image --exit-code 0 --severity HIGH,CRITICAL $service:latest

                done

            '''

        }

    }

    stage('Push Images') {

        steps {

            withCredentials([[

                $class: 'AmazonWebServicesCredentialsBinding',

                credentialsId: 'aws-credentials'

            ]]) {

                sh '''

                    aws ecr get-login-password --region $AWS_REGION | \

                    docker login --username AWS --password-stdin $ECR_REGISTRY

                    docker tag auth-service:latest \

                    $ECR_REGISTRY/auth-service:latest

                    docker push \

                    $ECR_REGISTRY/auth-service:latest

                    docker tag user-service:latest \

                    $ECR_REGISTRY/user-service:latest

                    docker push \

                    $ECR_REGISTRY/user-service:latest

                '''

            }

            withCredentials([usernamePassword(

                credentialsId: 'nexus-creds',

                usernameVariable: 'NEXUS_USER',

                passwordVariable: 'NEXUS_PASS'

            )]) {

                sh '''

                    echo "$NEXUS_PASS" | docker login \

                    $NEXUS_REGISTRY \

                    -u "$NEXUS_USER" \

                    --password-stdin

                    services="audit-service contact-service customer-service file-service
 
