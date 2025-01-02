pipeline {
    agent any
    environment {
        DOCKER_ID = "afarah1208"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
        DOCKER_CAST_IMAGE = "cast-service"
        DOCKER_MOVIE_IMAGE = "movie-service"
    }
    stages {
        stage('Docker Build') {
            steps {
                script {
                    sh '''
                    if (docker ps -a | grep -q cast-service); then
                        docker stop cast-service
                        docker rm -f cast-service
                    fi
                    if (docker ps -a | grep -q movie-service); then
                        docker stop movie-service
                        docker rm -f movie-service
                    fi
                    if (docker ps -a | grep -q cast-db); then
                        docker stop cast-db
                        docker rm -f cast-db
                    fi
                    if (docker ps -a | grep -q movie-db); then
                        docker stop movie-db
                        docker rm -f movie-db
                    fi
                    if (docker ps -a | grep -q nginx-service); then
                        docker stop nginx-service
                        docker rm -f nginx-service
                    fi
                    docker build -t $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG ./cast-service
                    docker build -t $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                    sleep 6
                    '''
                }
            }
        }
        stage('Docker run') {
            steps {
                script {
                    sh '''
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
                    docker run -d --name cast-service \
                        -p 8002:8000 \
                        -e DATABASE_URI=postgresql://cast_db_user:cast_db_password@cast-db:5432/cast_db \
                        $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                    docker run -d --name movie-service \
                        -p 8001:8000 \
                        -e DATABASE_URI=postgresql://movie_db_user:movie_db_password@movie-db:5432/movie_db \
                        -e CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ \
                        $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                    docker run -d --name nginx-service \
                        -p 80:8080 \
                        -v ./nginx_config.conf:/etc/nginx/conf.d/default.conf \
                        nginx:1.17.6-alpine
                    sleep 10
                    '''
                }
            }
        }
        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                    curl localhost
                    '''
                }
            }
        }
        stage('Docker Push') {
            steps {
                script {
                    sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                    docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                    '''
                }
            }

        }
        stage('Dev deployment') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    helm -n dev upgrade --install movie-db --values movie-db-helm/values.yaml movie-db-helm/
                    helm -n dev upgrade --install cast-db --values cast-db-helm/values.yaml cast-db-helm/
                    helm -n dev upgrade --install movie-service --values movie-helm/values.yaml --set image.repository=$DOCKER_ID/$DOCKER_MOVIE_IMAGE --set image.tag=$DOCKER_TAG movie-helm/
                    helm -n dev upgrade --install cast-service --values cast-helm/values.yaml --set image.repository=$DOCKER_ID/$DOCKER_CAST_IMAGE --set image.tag=$DOCKER_TAG cast-helm/
                    helm -n dev upgrade --install nginx-service --values nginx-helm/values.yaml nginx-helm/
                    '''
                }
            }
        }
        stage('QA deployment') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    helm -n qa upgrade --install movie-db --values movie-db-helm/values.yaml movie-db-helm/
                    helm -n qa upgrade --install cast-db --values cast-db-helm/values.yaml cast-db-helm/
                    helm -n qa upgrade --install movie-service --values movie-helm/values.yaml --set image.repository=$DOCKER_ID/$DOCKER_MOVIE_IMAGE --set image.tag=$DOCKER_TAG movie-helm/
                    helm -n qa upgrade --install cast-service  --values cast-helm/values.yaml --set image.repository=$DOCKER_ID/$DOCKER_CAST_IMAGE --set image.tag=$DOCKER_TAG cast-helm/
                    helm -n qa upgrade --install nginx-service --values nginx-helm/values.yaml --set service.nodePort=30082 nginx-helm/
                    '''
                }
            }
        }
        stage('Staging deployment') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    helm -n staging upgrade --install movie-db --values movie-db-helm/values.yaml movie-db-helm/
                    helm -n staging upgrade --install cast-db --values cast-db-helm/values.yaml cast-db-helm/
                    helm -n staging upgrade --install movie-service --values movie-helm/values.yaml --set image.repository=$DOCKER_ID/$DOCKER_MOVIE_IMAGE --set image.tag=$DOCKER_TAG movie-helm/
                    helm -n staging upgrade --install cast-service --values cast-helm/values.yaml --set image.repository=$DOCKER_ID/$DOCKER_CAST_IMAGE --set image.tag=$DOCKER_TAG cast-helm/
                    helm -n staging upgrade --install nginx-service --values nginx-helm/values.yaml --set service.nodePort=30084 nginx-helm/
                    '''
                }
            }
        }
        stage('Prod deployment') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                }
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    helm -n prod upgrade --install movie-db --values movie-db-helm/values.yaml movie-db-helm/
                    helm -n prod upgrade --install cast-db --values cast-db-helm/values.yaml cast-db-helm/
                    helm -n prod upgrade --install movie-service --values movie-helm/values.yaml --set image.repository=$DOCKER_ID/$DOCKER_MOVIE_IMAGE --set image.tag=$DOCKER_TAG movie-helm/
                    helm -n prod upgrade --install cast-service --values cast-helm/values.yaml --set image.repository=$DOCKER_ID/$DOCKER_CAST_IMAGE --set image.tag=$DOCKER_TAG cast-helm/
                    helm -n prod upgrade --install nginx-service --values nginx-helm/values.yaml --set service.nodePort=30086 nginx-helm/
                    '''
                }
            }
        }
    }
}
