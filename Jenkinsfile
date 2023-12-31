pipeline {
environment { // Declaration of environment variables
DOCKER_ID = "ykadi" // replace this with your docker-id
DOCKER_IMAGE = "datascientestapi"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
KUBECONFIG = credentials("EKS-config") // we retrieve  kubeconfig from secret file called config saved on jenkins
DOCKER_PASS = credentials("DOCKER_HUB_PASS")
AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
AWS_DEFAULT_REGION = "eu-west-3"
}
agent any // Jenkins will be able to select all available agents
stages {
        stage(' Docker Build'){ // docker build image stage
            steps {
                script {
                sh '''
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
                sleep 6
                '''
                }
            }
        }
        stage(' Docker run'){ // run container from our builded image
                steps {
                    script {
                    sh '''
                    echo "Cleaning existing container if exist"
                    docker ps -a | grep -i fastapi && docker rm -f fastapi
                    docker run -d -p 80:5000 --name fastapi $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                    sleep 10
                    '''
                    }
                }
            }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl -X GET -i http://0.0.0.0:80
                    '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            steps {
                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                '''
                }
            }

        }

        stage('Deploiement en dev'){
            steps {
                script {
                    sh '''
                    echo "Installation Ingress-controller Nginx"
                    helm upgrade --install ingress-nginx ingress-nginx \
	                --repo https://kubernetes.github.io/ingress-nginx \
	                --namespace ingress-nginx --create-namespace     
                    sleep 10

                    echo "Installation Cert-Manager"
                    helm upgrade --install cert-manager cert-manager \
                    --repo https://charts.jetstack.io \
                    --create-namespace --namespace cert-manager \
                    --set installCRDs=true
                    sleep 10
                        
                    echo "Installation Projet Devops 2023"
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" myapp1/values.yaml
                    helm upgrade --install myapp-release-dev myapp1/ --values myapp1/values.yaml -f myapp1/values-dev.yaml -n dev --create-namespace
                        
                    echo "Installation stack Prometheus-Grafana"
                    helm upgrade --install kube-prometheus-stack kube-prometheus-stack \
                    --namespace kube-prometheus-stack --create-namespace \
                    --repo https://prometheus-community.github.io/helm-charts
                    '''
                }
            }
                    

        }

        stage('Deploiement en staging') {
            steps {
               script {
                    sh '''
                    curl -i -i 'POST' -H 'Content-Type: application/json' -d '{"id": 1, "name": "toto", "email": "toto@email.com","password": "passwordtoto"}' https://www.devops-youss.cloudns.ph
                    if curl -i 'GET' -H 'accept: application/json' https://www.devops-youss.cloudns.ph/users | grep -qF "toto"; then
                        echo "La chaîne 'toto' a été trouvée dans la réponse."
                    else
                        echo "La chaîne 'toto' n'a pas été trouvée dans la réponse."
                    fi'''
                    def code_retour = sh '''curl -i -H 'accept: application/json' https://www.devops-youss.cloudns.ph/users/1 | grep -i "200"''' 
                    def count = sh '''curl 'GET' -H 'accept: application/json' https://www.devops-youss.cloudns.ph/users/1 ''' 
                    
                    if (code_retour == 200 ) {
                        echo $count
                    } else {
                        error("pas de retour de l'api")
                    }    
            }

        stage('Deploiement en prod'){
            steps {
                // Create an Approval Button with a timeout of 15minutes.
                // this require a manuel validation in order to deploy on production environment
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                }

                script {
                    sh '''
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" myapp1/values.yaml     
                    helm upgrade --install myapp-release-prod myapp1/ --values myapp1/values.yaml -f myapp1/values-prod.yaml -n prod --create-namespace
                    '''
                }
            }

        }

        stage('Prune Docker data') {
                steps {
                    sh 'docker system prune -a --volumes -f'
                }

        }
    }
        post { // send email when the job has failed
            success {
                script {
                    slackSend botUser: true, color: 'good', message: "Successful :jenkins-${JOB_NAME}-${BUILD_ID}", teamDomain: 'DEVOPS TEAM', tokenCredentialId: 'slack-bot-token'
                }
            }
            
            failure {
                script {
                    slackSend botUser: true, color: 'danger', message: "Failure :jenkins-${JOB_NAME}-${BUILD_ID}", teamDomain: 'DEVOPS TEAM', tokenCredentialId: 'slack-bot-token'
                }
            }
            // ..
            /*
            failure {
                echo "This will run if the job failed"
                mail to: "youssef.kadi@gmail.com",
                    subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
                    body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
            }
            */
            // ..
        }
    }