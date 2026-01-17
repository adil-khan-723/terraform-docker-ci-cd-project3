pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        AWS_ACCOUNT_ID = "736786104206"

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_REPOSITORY = "oggy-app-dev"

        ASSUME_ROLE_ARN = "arn:aws:iam::736786104206:role/ci-ecr-push-role-dev"
    }

    stages {
        stage ("Checkout source") {
            steps {
                git branch: 'main', changelog: false, poll: false, url: 'https://github.com/adil-khan-723/terraform-docker-application-code-project3.git'
            }
        }

        stage ("Get commit SHA") {
            steps {
                script {
                    env.COMMIT_SHA = sh(
                    script: 'git rev-parse --short HEAD',
                    returnStdout: true
                ).trim()
                }
                echo "Using image tag: ${env.COMMIT_SHA}"
            }
        }

        stage ("Assume role for ECR credentials") {
            steps {
                script {
                    def creds = sh (
                        script: """
                            aws sts assume-role \
                            --role-arn ${ASSUME_ROLE_ARN} \
                            --role-session-name jenkins-ci-session \
                            --output json
                        """,
                        returnStdout: true
                    ).trim()

                    def json = readJSON text: creds

                    env.AWS_ACCESS_KEY_ID     = json.Credentials.AccessKeyId
                    env.AWS_SECRET_ACCESS_KEY = json.Credentials.SecretAccessKey
                    env.AWS_SESSION_TOKEN     = json.Credentials.SessionToken
                }
            }
        }

        stage ("Authenticate to ECR") {
            steps {
                sh """ 
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login \
                    --username AWS \
                    --password-stdin ${ECR_REGISTRY}
                """
            }
        }

        stage ("Build docker images") {
            steps {
                sh """
                    docker build -t backend:${env.COMMIT_SHA} ./backend 
                    docker build -t frontend:${env.COMMIT_SHA} ./frontend
                """
            }
        }

        stage ("Tag Images for ECR") {
            steps {
                sh """
                    docker tag backend:${env.COMMIT_SHA} ${ECR_REGISTRY}/${ECR_REPOSITORY}:backend-${env.COMMIT_SHA}
                    docker tag frontend:${env.COMMIT_SHA} ${ECR_REGISTRY}/${ECR_REPOSITORY}:frontend-${env.COMMIT_SHA}
                """
            }
        }

        stage ("Push images to ECR") {
            steps {
                sh """
                    docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:backend-${env.COMMIT_SHA}
                    docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:frontend-${env.COMMIT_SHA}
                """
            }
        }
    }

    post {
    always {
      echo "CI completed for commit ${env.COMMIT_SHA}"
    }

    success {
      echo "Images pushed successfully to ECR"
    }

    failure {
      echo "CI failed — no images pushed"
    }
  }
}