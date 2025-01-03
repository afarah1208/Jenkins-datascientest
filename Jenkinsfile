pipeline {
    environment {
        DOCKER_ID = "afarah1208"
        DOCKER_IMAGE_CAST_SERVICE = "cast-service"
        DOCKER_IMAGE_MOVIE_SERVICE = "movie-service"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }
    agent any
    stages {
        stage('Docker Build') {
            steps {
                script {
                    sh '''
                    #!/bin/bash
                    for service in cast-service movie-service cast-db movie-db nginx-service; do
                        if (docker ps -a | grep -q $service); then
                            docker stop $service
                            docker rm -f $service
                        fi
                    done

                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE:$DOCKER_TAG ./cast-service
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE:$DOCKER_TAG ./movie-service
                    '''
                }
            }
        }

        stage('Docker Run') {
            steps {
                script {
                    sh '''
                    #!/bin/bash
                    docker run -d --name cast-db \
                        -v postgres_data_cast:/var/lib/postgresql/data/ \
                        -e POSTGRES_USER=cast_db_user \
                        -e POSTGRES_PASSWORD=cast_db_password \
                        -e POSTGRES_DB=cast_db \
                        postgres:12.1-alpine

                    docker run -d --name movie-db \
                        -v postgres_data_movie:/var/lib/postgresql/data/ \
                        -e POSTGRES_USER=movie_db_user \
                        -e POSTGRES_PASSWORD=movie_db_password \
                        -e POSTGRES_DB=movie_db \
                        postgres:12.1-alpine

                    sleep 10

                    docker run -d --name cast-service \
                        -p 8002:8000 \
                        -e DATABASE_URI=postgresql://cast_db_user:cast_db_password@cast-db:5432/cast_db \
                        $DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE:$DOCKER_TAG

                    docker run -d --name movie-service \
                        -p 8001:8000 \
                        -e DATABASE_URI=postgresql://movie_db_user:movie_db_password@movie-db:5432/movie_db \
                        -e CAST_SERVICE_HOST_URL=http://cast-service:8000/api/v1/casts/ \
                        $DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE:$DOCKER_TAG

                    docker run -d --name nginx-service \
                        -p 80:8080 \
                        -v ./nginx_config.conf:/etc/nginx/conf.d/default.conf \
                        nginx:1.17.6-alpine
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                    #!/bin/bash
                    endpoints=(
                        "http://localhost/api/v1/casts/"
                        "http://localhost/api/v1/movies/"
                    )

                    for endpoint in "${endpoints[@]}"; do
                        echo "Testing $endpoint"
                        response=$(curl -s -o /dev/null -w "%{http_code}" $endpoint)

                        if [ "$response" == "200" ]; then
                            echo "SUCCESS: $endpoint is reachable."
                        else
                            echo "FAILURE: $endpoint returned HTTP code $response."
                            exit 1
                        fi
                    done
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                    #!/bin/bash
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE:$DOCKER_TAG
                    docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE:$DOCKER_TAG
                    '''
                }
            }
        }

        // Déploiement dans dev, qa, staging, prod
        stage('Deployment in dev') {
            steps {
                deployWithHelm('dev')
            }
        }

        stage('Deployment in qa') {
            steps {
                deployWithHelm('qa')
            }
        }

        stage('Deployment in staging') {
            steps {
                deployWithHelm('staging')
            }
        }

        stage('Deployment in prod') {
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }
                deployWithHelm('prod')
            }
        }
    }
}

def deployWithHelm(env) {
    environment {
        KUBECONFIG = credentials("config")
    }
    sh """
    #!/bin/bash
    rm -Rf .kube
    mkdir .kube
    echo "$KUBECONFIG" > .kube/config

    helm upgrade --install moviecast-helm ./helm \
        --namespace ${env} --create-namespace \
        --values ./helm/values.yaml \
        --set namespace=${env} \
        --set cast_service.image=$DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE \
        --set cast_service.imageTag=$DOCKER_TAG \
        --set movie_service.image=$DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE \
        --set movie_service.imageTag=$DOCKER_TAG

    echo "Waiting for Nginx service in ${env} to be ready..."
    sleep 30

    NODE_PORT=$(kubectl get svc nginx-service -n ${env} -o jsonpath='{.spec.ports[0].nodePort}')
    NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

    echo "Nginx is accessible on: http://$NODE_IP:$NODE_PORT"
    echo "Movies API endpoint: http://$NODE_IP:$NODE_PORT/api/v1/movies/"
    echo "Casts API endpoint: http://$NODE_IP:$NODE_PORT/api/v1/casts/"
    """
}
