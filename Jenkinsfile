pipeline {
    agent any

    environment {
        IMAGE_REPO = 'ghcr.io/yujajja/simple-web'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Yujajja/k8s-cicd-lab.git'
            }
        }

        stage('Prepare Image Tag') {
            steps {
                script {
                    env.IMAGE_TAG = "sha-${env.GIT_COMMIT}"
                }

                echo "Image: ${IMAGE_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_REPO}:${IMAGE_TAG} \
                      -t ${IMAGE_REPO}:latest \
                      ./app
                '''
            }
        }

        stage('Push Image to GHCR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-ghcr',
                        usernameVariable: 'GH_USER',
                        passwordVariable: 'GH_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$GH_TOKEN" | docker login ghcr.io \
                          -u "$GH_USER" \
                          --password-stdin

                        docker push ${IMAGE_REPO}:${IMAGE_TAG}
                        docker push ${IMAGE_REPO}:latest

                        docker logout ghcr.io
                    '''
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-ghcr',
                        usernameVariable: 'GH_USER',
                        passwordVariable: 'GH_TOKEN'
                    )
                ]) {
                    sh '''
                        sed -i "s|image: .*simple-web.*|image: ${IMAGE_REPO}:${IMAGE_TAG}|g" deployment.yaml

                        git config user.name "jenkins"
                        git config user.email "jenkins@local"

                        git add deployment.yaml

                        if git diff --cached --quiet; then
                            echo "Manifest already uses this image."
                            exit 0
                        fi

                        git commit -m "Jenkins: update image to ${IMAGE_TAG} [skip ci]"

                        cat > /tmp/git-askpass.sh <<'EOF'
#!/bin/sh
case "$1" in
  *Username*) echo "$GH_USER" ;;
  *Password*) echo "$GH_TOKEN" ;;
esac
EOF

                        chmod 700 /tmp/git-askpass.sh

                        GIT_ASKPASS=/tmp/git-askpass.sh \
                        GIT_TERMINAL_PROMPT=0 \
                        git push origin HEAD:main

                        rm -f /tmp/git-askpass.sh
                    '''
                }
            }
        }
    }
}