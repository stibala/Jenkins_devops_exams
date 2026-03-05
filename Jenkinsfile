pipeline {
    environment {
        // Declaration of environment variables
        DOCKER_ID          = "adamstibal"                        // replace this with your docker hub username/id
        DOCKER_IMAGE_MOVIE = "dst-exam-movie-api"
        DOCKER_IMAGE_CAST  = "dst-exam-cast-api"
        DOCKER_TAG         = "v.${BUILD_ID}.0"                // tags increment with each build
    }

    agent any

    stages {
        stage('Build & Local Test') {
            steps {
                script {
                    // Build images using the correct context
                    sh "docker build -t ${DOCKER_ID}/${DOCKER_IMAGE_MOVIE}:${DOCKER_TAG} ./movie-service"
                    sh "docker build -t ${DOCKER_ID}/${DOCKER_IMAGE_CAST}:${DOCKER_TAG} ./cast-service"
                    
                    // Use docker compose for integration testing as it handles DB dependencies
                    sh "docker compose down || true"
                    sh "docker compose up -d --build"

                    // Wait for services to be ready
                    echo "Waiting for services to start..."
                    sleep 20
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    // Test via nginx proxy or directly
                    // Expecting 200 OK from movies (returns empty list if db empty)
                    sh "curl -f http://localhost:8001/api/v1/movies/ || (docker compose logs movie_service && exit 1)"
                    sh "curl -f -s -X POST -H \"Content-Type: application/json\" -d '{\"name\": \"a\", \"nationality\": \"b\"}' http://localhost:8002/api/v1/casts/ | grep -E '201' || (docker compose logs cast_service && exit 1)"
                    // Test nginx proxying
                    sh "curl -f http://localhost:8080/api/v1/movies/ || (docker compose logs nginx && exit 1)"
                }
            }
            post {
                always {
                    sh "docker compose down"
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh "echo \"$DOCKER_PASS\" | docker login -u ${DOCKER_ID} --password-stdin"
                    sh "docker push ${DOCKER_ID}/${DOCKER_IMAGE_MOVIE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_ID}/${DOCKER_IMAGE_CAST}:${DOCKER_TAG}"
                }
            }
        }

        stage('Deployment in dev') {
            steps {
                deployToEnv('dev', 30081, 30082)
            }
        }

        stage('Deployment in QA') {
            steps {
                deployToEnv('qa', 30083, 30084)
            }
        }

        stage('Deployment in staging') {
            steps {
                deployToEnv('staging', 30085, 30086)
            }
        }

        stage('Deployment in prod') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you want to deploy in production?',
                          ok:      'Yes, deploy to production'
                }
                deployToEnv('prod', 30087, 30088)
            }
        }
    }

    post {
        always {
            echo "Pipeline finished - Build #${BUILD_ID}"
        }
    }
}

// Helper function for Helm deployment
def deployToEnv(envName, moviePort, castPort) {
    // Assuming databases are already deployed or managed separately
    // The URIs use the naming convention of the release and namespace
    def dbUser = "movie_db_username"
    def dbPass = "movie_db_password"
    def movieDbUri = "postgresql://${dbUser}:${dbPass}@movie-db-${envName}:5432/movie_db_${envName}"
    def castDbUri = "postgresql://${dbUser}:${dbPass}@cast-db-${envName}:5432/cast_db_${envName}"
    def castSvcUrl = "http://cast-api-${envName}:80/api/v1/casts/"

    withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG_FILE')]) {
        // Deploy Cast Service
        sh """
            helm upgrade --install cast-api-${envName} ./charts \
                --kubeconfig $KUBECONFIG_FILE \
                --namespace ${envName} \
                --create-namespace \
                --set image.repository=${DOCKER_ID}/${DOCKER_IMAGE_CAST} \
                --set image.tag=${DOCKER_TAG} \
                --set service.nodePort=${castPort} \
                --set probePath=/api/v1/casts/ \
                --set databaseUri=${castDbUri}
        """

        // Deploy Movie Service
        sh """
            helm upgrade --install movie-api-${envName} ./charts \
                --kubeconfig $KUBECONFIG_FILE \
                --namespace ${envName} \
                --create-namespace \
                --set image.repository=${DOCKER_ID}/${DOCKER_IMAGE_MOVIE} \
                --set image.tag=${DOCKER_TAG} \
                --set service.nodePort=${moviePort} \
                --set probePath=/api/v1/movies/ \
                --set databaseUri=${movieDbUri} \
                --set castServiceHostUrl=${castSvcUrl}
        """
    }
}
