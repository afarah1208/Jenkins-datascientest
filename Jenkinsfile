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

stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp helm/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install moviecast-helm ./helm \
        --namespace dev --create-namespace \
        --values ./helm/values.yaml \
        --set namespace=dev \
        --set cast_service.image.repository=${DOCKER_ID}/${DOCKER_IMAGE_CAST_SERVICE}:${DOCKER_TAG} \
        --set movie_service.image.repository=${DOCKER_ID}/${DOCKER_IMAGE_MOVIE_SERVICE}:${DOCKER_TAG}

    echo "Waiting for Nginx service in dev to be ready..."
    sleep 15

    NODE_PORT=\$(kubectl get svc nginx -n dev -o jsonpath='{.spec.ports[0].nodePort}')
    echo "Nginx is accessible on: http://NODE_IP:\$NODE_PORT"
    echo "Movies API endpoint: http://NODE_IP:\$NODE_PORT/api/v1/movies/"
    echo "Casts API endpoint: http://NODE_IP:\$NODE_PORT/api/v1/casts/"
    '''
                }
            }

        }
stage('Deploiement en qa'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp helm/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install moviecast-helm ./helm \
        --namespace qa --create-namespace \
        --values ./helm/values.yaml \
        --set namespace=qa \
        --set cast_service.image=${DOCKER_ID}/${DOCKER_IMAGE_CAST_SERVICE} \
        --set cast_service.imageTag=${DOCKER_TAG} \
        --set movie_service.image=${DOCKER_ID}/${DOCKER_IMAGE_MOVIE_SERVICE} \
        --set movie_service.imageTag=${DOCKER_TAG}

    echo "Waiting for Nginx service in qa to be ready..."
        sleep 15

    NODE_PORT=\$(kubectl get svc nginx -n qa -o jsonpath='{.spec.ports[0].nodePort}')
    echo "Nginx is accessible on: http://NODE_IP:\$NODE_PORT"
    echo "Movies API endpoint: http://NODE_IP:\$NODE_PORT/api/v1/movies/"
    echo "Casts API endpoint: http://NODE_IP:\$NODE_PORT/api/v1/casts/"'''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp helm/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install moviecast-helm ./helm \
        --namespace staging --create-namespace \
        --values ./helm/values.yaml \
        --set namespace=staging \
        --set cast_service.image=${DOCKER_ID}/${DOCKER_IMAGE_CAST_SERVICE} \
        --set cast_service.imageTag=${DOCKER_TAG} \
        --set movie_service.image=${DOCKER_ID}/${DOCKER_IMAGE_MOVIE_SERVICE} \
        --set movie_service.imageTag=${DOCKER_TAG}

    echo "Waiting for Nginx service in staging to be ready..."
        sleep 15

    NODE_PORT=\$(kubectl get svc nginx -n staging -o jsonpath='{.spec.ports[0].nodePort}')
    echo "Nginx is accessible on: http://NODE_IP:\$NODE_PORT"
    echo "Movies API endpoint: http://NODE_IP:\$NODE_PORT/api/v1/movies/"
    echo "Casts API endpoint: http://NODE_IP:\$NODE_PORT/api/v1/casts/"'''
                }
            }

        }
  stage('Deploiement en prod'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
            // Create an Approval Button with a timeout of 15minutes.
            // this require a manuel validation in order to deploy on production environment
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }

                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp helm/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install moviecast-helm ./helm \
        --namespace prod --create-namespace \
        --values ./helm/values.yaml \
        --set namespace=prod \
        --set cast_service.image=${DOCKER_ID}/${DOCKER_IMAGE_CAST_SERVICE} \
        --set cast_service.imageTag=${DOCKER_TAG} \
        --set movie_service.image=${DOCKER_ID}/${DOCKER_IMAGE_MOVIE_SERVICE} \
        --set movie_service.imageTag=${DOCKER_TAG}

    echo "Waiting for Nginx service in prod to be ready..."
       sleep 15

    NODE_PORT=\$(kubectl get svc nginx -n prod -o jsonpath='{.spec.ports[0].nodePort}')
    echo "Nginx is accessible on: http://NODE_IP:\$NODE_PORT"
    echo "Movies API endpoint: http://NODE_IP:\$NODE_PORT/api/v1/movies/"
    echo "Casts API endpoint: http://NODE_IP:\$NODE_PORT/api/v1/casts/"
                '''
                }
            }

        }

}
}