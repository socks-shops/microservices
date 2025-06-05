pipeline {
    environment {
        AWS_REGION = 'us-east-1'
        CLUSTER_NAME = 'sockshop-EKS'
        MONGODB_OPERATOR_CHART_NAME = 'helm-charts/operators/mongodb/'
        MICROSERVICES_CHART_NAME = 'helm-charts/other-microservices/'
        NAMESPACE = 'dev'
        RELEASE_NAME = 'socksshop-microservices'
    }
    agent any

    stages {


        stage('Define Namespace') {
            steps {
                script {
                    sh'''
                    printenv
                    '''
                    def currentBranch = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                    def currentBranch = env.BRANCH_NAME // Jenkins variable

                    if (currentBranch == 'main') {
                        env.NAMESPACE = 'dev'
                    } else if (currentBranch == 'staging') {
                        env.NAMESPACE = 'staging'
                    } else if (currentBranch == 'prod') {
                        env.NAMESPACE = 'prod'
                    } else {
                        error "Branche '${currentBranch}' non gérée pour la définition du namespace."
                    }

                    echo "Le namespace défini pour cette exécution est : ${env.NAMESPACE}"
                }
            }
        }

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

                                    velero backup create microservices-pre-deploy-backup-${NAMESPACE}-${BUILD_NUMBER} --include-namespaces=${NAMESPACE}
                                    '''
                                }
                            }
                        }
                    }
                }

            }
        }

        stage('${NAMESPACE} : Confirm deployment') {
            when {
                environment name: 'NAMESPACE', value: 'prod'
            }
            steps {
                script {
                    input message: "Continuer le déploiement en ${env.NAMESPACE} ? Cliquez sur 'Proceed' pour confirmer.",
                        ok: 'Proceed'
                }
            }
        }

        stage('${NAMESPACE} : Microservices Deployment - AWS EKS') {
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

                    helm repo add percona https://percona.github.io/percona-helm-charts/
                    helm repo update
                    helm upgrade --install carts-db percona/psmdb-db -n ${NAMESPACE} -f ${MONGODB_OPERATOR_CHART_NAME}${NAMESPACE}-cart-db-values.yaml
                    helm upgrade --install orders-db percona/psmdb-db -n ${NAMESPACE} -f ${MONGODB_OPERATOR_CHART_NAME}${NAMESPACE}-orders-db-values.yaml
                    helm upgrade --install user-db percona/psmdb-db -n ${NAMESPACE} -f ${MONGODB_OPERATOR_CHART_NAME}${NAMESPACE}-user-db-values.yaml

                    rm -Rf helm-charts
                    git clone https://${GITHUB_USERNAME}:${GITHUB_PASSWORD}@github.com/socks-shops/helm-charts.git helm-charts
                    helm upgrade --install ${RELEASE_NAME} ${MICROSERVICES_CHART_NAME} -n ${NAMESPACE}

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
                            helm uninstall $RELEASE_NAME --namespace $NAMESPACE --wait
                            helm uninstall carts-db -n $NAMESPACE --wait
                            helm uninstall orders-db -n $NAMESPACE --wait
                            helm uninstall user-db -n $NAMESPACE --wait
                            kubectl get all -n $NAMESPACE
                            '''
                        } else if (userChoice == 'Rollback') {
                            echo "Rollback du déploiement..."
                            sh '''
                            helm rollback $RELEASE_NAME 0 --wait --namespace $NAMESPACE || echo "Aucune révision précédente disponible pour rollback de ${RELEASE_NAME}"
                            helm rollback carts-db 0 --wait --namespace $NAMESPACE || echo "Aucune révision précédente disponible pour rollback de carts-db"
                            helm rollback orders-db 0 --wait --namespace $NAMESPACE || echo "Aucune révision précédente disponible pour rollback de orders-db"
                            helm rollback user-db 0 --wait --namespace $NAMESPACE || echo "Aucune révision précédente disponible pour rollback de user-db"
                            kubectl get all -n $NAMESPACE
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
