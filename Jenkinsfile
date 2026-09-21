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

                    for service in $services; do

                        echo "Building $service"

                        docker build -t $service:latest -f ./$service/Dockerfile .

                    done

                '''

            }

        }

        stage('Trivy Image Scan') {

            steps {

                sh '''

                    services="auth-service user-service audit-service contact-service customer-service file-service gateway-service invoice-service"

                    for service in $services; do

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

                        TAG=${BUILD_NUMBER}

                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

                        for service in auth-service user-service; do

                            docker tag $service:latest $ECR_REGISTRY/$service:$TAG

                            docker push $ECR_REGISTRY/$service:$TAG

                        done

                    '''

                }

                withCredentials([usernamePassword(

                    credentialsId: 'nexus-creds',

                    usernameVariable: 'NEXUS_USER',

                    passwordVariable: 'NEXUS_PASS'

                )]) {

                    sh '''

                        echo "$NEXUS_PASS" | docker login $NEXUS_REGISTRY -u "$NEXUS_USER" --password-stdin

                        services="audit-service contact-service customer-service file-service gateway-service invoice-service"

                        for service in $services; do

                            echo "Pushing $service to Nexus"

                            docker tag $service:latest $NEXUS_REGISTRY/$service:latest

                            docker push $NEXUS_REGISTRY/$service:latest

                        done

                    '''

                }

            }

        }

        stage('EKS Authentication') {

            steps {

                sh '''

                    export AWS_PAGER=""

                    echo "Checking AWS credentials..."

                    aws sts get-caller-identity

                    echo "Updating EKS kubeconfig..."

                    aws eks update-kubeconfig --region us-east-1 --name speshway-live-dev-eks

                    echo "Checking EKS nodes..."

                    kubectl get nodes

                '''

            }

        }

        stage('Helm Deploy') {

            steps {

                sh '''

                    echo "Deploying application using Helm"

                    if [ -d helm ]; then

                        helm upgrade --install speshway helm/ crm \

                            --namespace dev \

                            --create-namespace

                    else

                        echo "Helm directory not found - deployment skipped"

                    fi

                '''

            }

        }

        stage('Rollout Status') {

            steps {

                sh '''

                    kubectl rollout status deployment \

                        --all \

                        --namespace dev \

                        --timeout=180s

                '''

            }

        }

        stage('Smoke Test') {

            steps {

                sh '''

                    echo "Running smoke test..."

                    kubectl get pods -n dev

                    kubectl get svc -n dev

                '''

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

        always {

            echo 'Pipeline execution completed.'

        }

    }

}
 
