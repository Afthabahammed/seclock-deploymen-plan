
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPOSITORY = 'seclock'
        AWS_ACCOUNT_ID = '268666184272'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate

                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pytest httpx
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    python -m pytest test_e2e.py -v
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQube'

                    withSonarQubeEnv('sonarqube') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} |
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-gitops',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        set -eu

                        ASKPASS_SCRIPT="$(mktemp)"
                        trap 'rm -f "$ASKPASS_SCRIPT"' EXIT
                        chmod 700 "$ASKPASS_SCRIPT"

                        cat > "$ASKPASS_SCRIPT" <<'EOF'
#!/bin/sh
case "$1" in
    *Username*) printf '%s\\n' "$GIT_USERNAME" ;;
    *Password*) printf '%s\\n' "$GIT_TOKEN" ;;
    *) exit 1 ;;
esac
EOF

                        export GIT_ASKPASS="$ASKPASS_SCRIPT"
                        export GIT_TERMINAL_PROMPT=0

                        git config user.name "Jenkins GitOps"
                        git config user.email "jenkins-gitops@example.com"

                        sed -i "s|image: .*dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" k8s/deployment.yaml

                        git add k8s/deployment.yaml

                        if git diff --cached --quiet; then
                            echo "No GitOps manifest changes to commit."
                        else
                            git commit -m "Deploy Seclock image ${IMAGE_TAG}"
                            git push \
                                "https://github.com/Afthabahammed/seclock-deploymen-plan.git" \
                                HEAD:main
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                rm -rf venv || true
            '''
        }
    }
}

