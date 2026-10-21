
pipeline {
    agent any

    environment {
        IMAGE_NAME = "stelabiju/takecare-devsecops"
        GIT_URL = "https://github.com/stelabiju/takecare.git"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'Github_jenkins',
                    url: "${GIT_URL}"
            }
        }

        stage('SAST Scan') {
            steps {
                sh '''
                    docker run --rm \
                    -v "$WORKSPACE:/src" \
                    semgrep/semgrep semgrep scan \
                    --config auto /src
                '''
            }
        }

        stage('Dependency and Config Scan') {
            steps {
                sh '''
                    docker run --rm \
                    -v "$WORKSPACE:/src" \
                    aquasec/trivy:latest fs \
                    --scanners vuln,misconfig /src
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $IMAGE_NAME:$BUILD_NUMBER .
                    docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" --password-stdin

                        docker push $IMAGE_NAME:$BUILD_NUMBER
                        docker push $IMAGE_NAME:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'Github_jenkins',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh '''
                        sed -i "s|image: .*|image: $IMAGE_NAME:$BUILD_NUMBER|" manifest/deployment.yaml

                        git config user.name "stelabiju"
                        git config user.email "stelabiju@gmail.com"

                        git add manifest/deployment.yaml

                        git commit -m "Update TakeCare image to build $BUILD_NUMBER"

                        git push https://$GIT_USER:$GIT_PASS@github.com/stelabiju/takecare.git HEAD:main
                    '''
                }
            }
        }
    }
}
