pipeline {
    environment {
        AWS_REGION = 'us-east-1'
        CLUSTER_NAME = 'sockshop-EKS'
        CHART_NAME = 'helm-charts/other-microservices/'
        NAMESPACE = 'dev'
        RELEASE_NAME = 'socksshop-microservices'
    }
    agent any

    stages {

        stage('Pre-Deployment Backups') {
            parallel {

                stage('Backup cluster') {
                    agent {
                        docker {
                            image 'socksshop/aws-cli-velero:latest'
                            args '-u root -v $HOME/.kube:/root/.kube'
                        }
                    }
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                                    sh '''
                                    aws eks --region $AWS_REGION update-kubeconfig --name $CLUSTER_NAME
                                    chmod 600 /root/.kube/config

                                    velero backup create microservices-pre-deploy-backup-${BUILD_NUMBER} --include-namespaces=dev
                                    '''
                                }
                            }
                        }
                    }
                }

            }
        }

        stage('Deploy to AWS EKS') {
            agent {
                docker {
                    image 'socksshop/aws-cli-git-kubectl-helm:latest'
                    args '-u root -v $HOME/.kube:/root/.kube'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    sh '''
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${CLUSTER_NAME}
                    kubectl config current-context

                    kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}

                    rm -Rf helm-charts
                    git clone https://${GITHUB_USERNAME}:${GITHUB_PASSWORD}@github.com/socks-shops/helm-charts.git helm-charts

                    helm upgrade --install ${RELEASE_NAME} ${CHART_NAME} --namespace ${NAMESPACE}

                    kubectl get all -n ${NAMESPACE}
                    '''
                }
            }
        }

        stage('Post-deployment managment') {
            agent {
                docker {
                    image 'socksshop/aws-cli-git-kubectl-helm:latest'
                    args '-u root -v $HOME/.kube:/root/.kube'
                }
            }
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    script {
                        def userChoice = input(
                            id: 'userInput', message: 'Que souhaitez-vous faire avec le déploiement EKS des microservices?',
                            parameters: [
                                choice(
                                    name: 'ACTION',
                                    choices: ['Continuer', 'Rollback', 'Destroy'],
                                    description: 'Choisissez une action à effectuer'
                                )
                            ]
                        )

                        sh '''
                        aws eks --region $AWS_REGION update-kubeconfig --name $CLUSTER_NAME
                        chmod 600 /root/.kube/config
                        kubectl config current-context
                        '''

                        if (userChoice == 'Destroy') {
                            echo "Destruction du déploiement..."
                            sh '''
                            helm uninstall $RELEASE_NAME --wait --namespace $NAMESPACE
                            kubectl get all -n $NAMESPACE
                            '''
                        } else if (userChoice == 'Rollback') {
                            echo "Rollback du déploiement..."
                            sh '''
                            helm rollback $RELEASE_NAME 0 --wait --namespace $NAMESPACE || echo "Aucune révision précédente disponible pour rollback"
                            '''
                        } else {
                            echo "Aucune action effectuée."
                        }
                    }
                }
            }
        }

    }
}
