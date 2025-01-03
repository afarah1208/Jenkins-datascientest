pipeline {
    environment {
        // Declaration of environment variables
        DOCKER_ID = "afarah1208" // replace this with your docker-id
        DOCKER_IMAGE_MOVIE_SERVICE = "movie-service" 
        DOCKER_IMAGE_CAST_SERVICE = "cast-service"
        DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
        DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
        KUBECONFIG = credentials("config") // we retrieve kubeconfig from secret file called config saved on jenkins
    }
    
    agent any // Jenkins will be able to select all available agents
    
    stages {
        stage('Docker Build') { // docker build image stage
            steps {
                script {
                    sh '''
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE:$DOCKER_TAG ./movie-service/
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE:$DOCKER_TAG ./cast-service/
                        sleep 6
                    '''
                }
            }
        }

        stage('Docker Run') { // run container from our built image
            steps {
                script {
                    sh '''
                        docker run -d -p 8088:80 --name jenkins_$BUILD_ID $DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE:$DOCKER_TAG
                        sleep 10
                    '''
                }
            }
        }

        /*stage('Test Acceptance') { // we launch the curl command to validate that the container responds to the request
            steps {
                script {
                    sh '''
                        curl localhost:8081
                    '''
                }
            }
        }*/

        stage('Docker Push') { // we pass the built image to our docker hub account
            steps {
                script {
                    sh '''
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE_SERVICE:$DOCKER_TAG
                        docker push $DOCKER_ID/$DOCKER_IMAGE_CAST_SERVICE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config")
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp moviecast_api_helm/values.yaml values.yml
                cat values.yml
                helm upgrade --install moviecast-api moviecast_api_helm --values=values.yml --namespace dev
                '''
                }
            }

        }
stage('Deploiement en qa'){
        environment
        {
        KUBECONFIG = credentials("config")
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp moviecast_api_helm/values.yaml values.yml
                cat values.yml
                helm upgrade --install moviecast-api moviecast_api_helm --values=values.yml --namespace qa
                '''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config")
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp moviecast_api_helm/values.yaml values.yml
                cat values.yml
                helm upgrade --install moviecast-api moviecast_api_helm --values=values.yml --namespace staging
                '''
                }
            }

        }
stage('Deploiement en prod'){
        environment
        {
          KUBECONFIG = credentials("config")
        }
        when {
          expression {
            return BRANCH_NAME ==~ /(origin\/main|origin\/master)/
          }
        }
            steps {
                timeout(time: 15, unit: "MINUTES") {
                  input message: 'Do you want to deploy in production ?', ok: 'Yes'
                }
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp moviecast_api_helm/values.yaml values.yml
                cat values.yml
                helm upgrade --install moviecast-api moviecast_api_helm --values=values.yml --namespace prod
                '''
                }
            }

        }

}
}