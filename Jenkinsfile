pipeline {
environment { // Declaration of environment variables
DOCKER_ID = "afarah1208" // replace this with your docker-id
DOCKER_IMAGE_CAST_SERVICE = "cast-service"
DOCKER_IMAGE_MOVIE_SERVICE = "movie-service"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
}
agent any // Jenkins will be able to select all available agents
stages {
        stage(' Docker Build'){ // docker build image stage
            steps {
                script {
                sh '''
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
        stage('Docker run'){ // run container from our builded image
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

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
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
                        fi
                    done
                    '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
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
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install moviecast-helm ./helm --namespace dev --create-namespace --values ./helm/values.yaml --set namespace=dev --set cast_service.image=$DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE --set cast_service.imageTag=$DOCKER_TAG --set movie_service.image=$DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE --set movie_service.imageTag=$DOCKER_TAG
                sleep 30
                NODE_PORT=$(kubectl get svc nginx-service -n prod -o jsonpath='{.spec.ports[0].nodePort}')
                echo "Nginx is accessible on port: $NODE_PORT"
                echo "Movies API endpoint: http://<node-ip>:$NODE_PORT/api/v1/movies/"
                echo "Casts API endpoint: http://<node-ip>:$NODE_PORT/api/v1/casts/"
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
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install moviecast-helm ./helm --namespace qa --create-namespace --values ./helm/values.yaml --set namespace=qa --set cast_service.image=$DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE --set cast_service.imageTag=$DOCKER_TAG --set movie_service.image=$DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE --set movie_service.imageTag=$DOCKER_TAG
                sleep 30
                NODE_PORT=$(kubectl get svc nginx-service -n prod -o jsonpath='{.spec.ports[0].nodePort}')
                echo "Nginx is accessible on port: $NODE_PORT"
                echo "Movies API endpoint: http://<node-ip>:$NODE_PORT/api/v1/movies/"
                echo "Casts API endpoint: http://<node-ip>:$NODE_PORT/api/v1/casts/"
                '''
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
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install moviecast-helm ./helm --namespace staging --create-namespace --values ./helm/values.yaml --set namespace=staging --set cast_service.image=$DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE --set cast_service.imageTag=$DOCKER_TAG --set movie_service.image=$DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE --set movie_service.imageTag=$DOCKER_TAG
                sleep 30
                NODE_PORT=$(kubectl get svc nginx-service -n prod -o jsonpath='{.spec.ports[0].nodePort}')
                echo "Nginx is accessible on port: $NODE_PORT"
                echo "Movies API endpoint: http://<node-ip>:$NODE_PORT/api/v1/movies/"
                echo "Casts API endpoint: http://<node-ip>:$NODE_PORT/api/v1/casts/"
                '''
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
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml                
                helm upgrade --install moviecast-helm ./helm --namespace prod --create-namespace --values ./helm/values.yaml --set namespace=prod --set cast_service.image=$DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE --set cast_service.imageTag=$DOCKER_TAG --set movie_service.image=$DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE --set movie_service.imageTag=$DOCKER_TAG
                sleep 30
                NODE_PORT=$(kubectl get svc nginx-service -n prod -o jsonpath='{.spec.ports[0].nodePort}')
                echo "Nginx is accessible on port: $NODE_PORT"
                echo "Movies API endpoint: http://<node-ip>:$NODE_PORT/api/v1/movies/"
                echo "Casts API endpoint: http://<node-ip>:$NODE_PORT/api/v1/casts/"
                '''
                }
            }

        }

}
}