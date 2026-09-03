pipeline {
    agent any

    environment {
        IMAGE_REPO = "agnimanu/productcatalogservice"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git(
                    url: 'https://github.com/Manoj-agni/productcatalogservice.git',
                    branch: 'main'
                )

                script {
                    env.IMAGE_NAME = "${IMAGE_REPO}:${env.GIT_COMMIT}"
                }
            }
        }

        stage('Check Environment') {
            steps {
                sh '''
                    echo "=== Go Version ==="
                    go version

                    echo "=== GCC Version ==="
                    gcc --version

                    echo "=== C Headers ==="
                    test -f /usr/include/stdlib.h
                    test -f /usr/include/stdio.h
                    test -f /usr/include/pthread.h

                    echo "Environment check successful"
                '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh '''
                    go clean -cache
                    go test ./...
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t "$IMAGE_NAME" .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | \
                        docker login \
                        --username "$DOCKER_USERNAME" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh '''
                    docker push "$IMAGE_NAME"
                '''
            }
        }

        stage('Update GitOps Deployment') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-creds',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e

                        rm -rf gitops

                        git clone \
                            "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Manoj-agni/GitOps.git" \
                            gitops

                        cd gitops/base/productcatalogservice/

                        git config user.email "jenkins@ci.com"
                        git config user.name "jenkins"

                        echo "Updating image to: $IMAGE_NAME"

                        sed -i \
                            "s|image: .*productcatalogservice.*|image: ${IMAGE_NAME}|g" \
                            deployment.yaml

                        echo "Updated deployment.yaml:"
                        grep "image:" deployment.yaml

                        git add deployment.yaml

                        git diff --cached

                        git commit \
                            -m "Update productcatalogservice image to ${IMAGE_NAME}" \
                            || echo "No changes to commit"

                        git push origin main
                    '''
                }
            }
        }
    }

    post {

        always {
            sh '''
                docker rmi "$IMAGE_NAME" || true
                docker logout || true
            '''
        }

        success {
            echo "Build, test, Docker push and GitOps update successful."
            echo "Image: ${IMAGE_NAME}"
        }

        failure {
            echo "Pipeline failed. Check the logs above."
        }
    }
}
