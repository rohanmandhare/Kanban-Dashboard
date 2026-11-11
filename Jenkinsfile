pipeline {
    agent any
    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        IMAGE_NAME = 'kanban-dashboard'
        CONTAINER_NAME = 'kanban-dashboard'
        APP_PORT = '80'
        CONTAINER_PORT = '8080'
        HEALTH_URL = 'http://127.0.0.1/healthz'
        // Replace with your Docker Hub namespace before running.
        DOCKERHUB_USER = 'rohan.ma2412@gmail.com'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHA = sh(script: "git rev-parse --short=12 HEAD", returnStdout: true).trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
                    env.FULL_IMAGE = "${env.DOCKERHUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build and Validate') {
            steps {
                sh '''
                    set -eu
                    test -f package.json
                    docker build --pull -t "$FULL_IMAGE" -t "$DOCKERHUB_USER/$IMAGE_NAME:build-$BUILD_NUMBER" .
                    docker image inspect "$FULL_IMAGE" >/dev/null
                    echo "Built image: $FULL_IMAGE"
                '''
            }
        }

        stage('Security Checks') {
            steps {
                sh '''
                    set -eu
                    IMAGE_USER="$(docker image inspect "$FULL_IMAGE" --format '{{.Config.User}}')"
                    test "$IMAGE_USER" != "" && test "$IMAGE_USER" != "0" && test "$IMAGE_USER" != "root"
                    echo "Non-root image user: $IMAGE_USER"
                '''
                // Optional: install Trivy on the Jenkins agent and uncomment:
                // sh 'trivy image --exit-code 1 --severity CRITICAL,HIGH "$FULL_IMAGE"'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                    usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_TOKEN')]) {
                    sh '''
                        set +x
                        echo "$REGISTRY_TOKEN" | docker login -u "$REGISTRY_USER" --password-stdin
                        docker push "$FULL_IMAGE"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    set -eu
                    OLD_IMAGE=""
                    if docker container inspect "$CONTAINER_NAME" >/dev/null 2>&1; then
                        OLD_IMAGE="$(docker inspect --format='{{.Config.Image}}' "$CONTAINER_NAME")"
                        docker stop "$CONTAINER_NAME" || true
                        docker rm "$CONTAINER_NAME" || true
                    fi
                    printf '%s' "$OLD_IMAGE" > .previous-image
                    docker pull "$FULL_IMAGE"
                    docker run -d --name "$CONTAINER_NAME" \\
                      --restart unless-stopped --memory=384m --cpus=1.0 \\
                      -p "$APP_PORT:$CONTAINER_PORT" "$FULL_IMAGE"

                    for i in $(seq 1 30); do
                        STATUS="$(docker inspect --format='{{.State.Health.Status}}' "$CONTAINER_NAME" 2>/dev/null || true)"
                        [ "$STATUS" = "healthy" ] && break
                        [ "$STATUS" = "unhealthy" ] && { docker logs "$CONTAINER_NAME"; exit 1; }
                        sleep 2
                    done
                    STATUS="$(docker inspect --format='{{.State.Health.Status}}' "$CONTAINER_NAME")"
                    test "$STATUS" = "healthy"
                    curl --fail --silent --show-error --max-time 5 "$HEALTH_URL" | grep -q ok
                    docker ps --filter "name=$CONTAINER_NAME"
                '''
            }
            post {
                failure {
                    sh '''
                        set +e
                        docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1
                        OLD_IMAGE="$(cat .previous-image 2>/dev/null)"
                        if [ -n "$OLD_IMAGE" ]; then
                            docker run -d --name "$CONTAINER_NAME" \\
                              --restart unless-stopped --memory=384m --cpus=1.0 \\
                              -p "$APP_PORT:$CONTAINER_PORT" "$OLD_IMAGE"
                            for i in $(seq 1 30); do
                                STATUS="$(docker inspect --format='{{.State.Health.Status}}' "$CONTAINER_NAME" 2>/dev/null)"
                                [ "$STATUS" = "healthy" ] && break
                                sleep 2
                            done
                            curl --fail --silent --show-error --max-time 5 "$HEALTH_URL" >/dev/null
                            echo "Rollback attempted to previous image: $OLD_IMAGE"
                        else
                            echo "No previous image recorded; no rollback target available."
                        fi
                    '''
                }
                always {
                    sh 'rm -f .previous-image'
                }
            }
        }

        stage('Post-deployment Validation') {
            steps {
                sh '''
                    set -eu
                    docker ps --filter "name=$CONTAINER_NAME" --format '{{.Names}} {{.Status}}'
                    docker inspect "$CONTAINER_NAME" --format='User={{.Config.User}} Health={{.State.Health.Status}} Memory={{.HostConfig.Memory}} NanoCPUs={{.HostConfig.NanoCpus}}'
                    curl --fail --silent --show-error --max-time 5 "$HEALTH_URL"
                    echo "Deployment validation passed."
                '''
            }
        }
    }

    post {
        success { echo "Deployment succeeded: ${env.FULL_IMAGE}" }
        failure { echo 'Pipeline failed. Review stage logs; rollback was attempted if a previous image existed.' }
    }
}
