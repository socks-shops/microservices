pipeline {
    environment {
        AWS_REGION = 'us-east-1'
        CLUSTER_NAME = 'sockshop-eks'
        MONGODB_OPERATOR_CHART_NAME = 'helm-charts/operators/mongodb/'
        MICROSERVICES_CHART_NAME = 'helm-charts/other-microservices/'
        // NAMESPACE = ''
        RELEASE_NAME = 'socksshop-microservices'
    }
    agent any

    stages {


        stage('Define Namespace') {
            steps {
                script {
                    checkout scm

                    def currentBranch = scm.branches[0].name
                    def namespace = ''

                    echo "${currentBranch}"

                    if (currentBranch == '*/main') {
                        namespace = 'dev'
                    } else if (currentBranch == '*/staging') {
                        namespace = 'staging'
                    } else if (currentBranch == '*/prod') {
                        namespace = 'prod'
                    } else {
                        error "Branche '${currentBranch}' non gérée pour la définition du namespace."
                    }

                    echo "${namespace}"
                    env.NAMESPACE = namespace
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

        stage('Confirm Microservices deployment - AWS EKS - Production') {
            when {
                environment name: 'NAMESPACE', value: 'prod'
            }
            steps {
                script {
                    input message: "Vous vous apprêtez à déployer dans l’environnement de ${env.NAMESPACE}, voulez-vous continuer ? Cliquez sur 'Proceed' pour confirmer.",
                        ok: 'Proceed'
                }
            }
        }

        stage('Microservices Deployment - AWS EKS') {
            agent {
                docker {
                    image 'socksshop/aws-cli-git-kubectl-helm-helm-secrets-age-sops:latest'
                    args '-u root -v $HOME/.kube:/root/.kube'
                }
            }
            environment {
                SOPS_AGE_KEY = credentials('AGE-SECRET-KEY')
            }
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    sh '''
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${CLUSTER_NAME}
                    kubectl config current-context

                    rm -Rf helm-charts
                    git clone https://${GITHUB_USERNAME}:${GITHUB_PASSWORD}@github.com/socks-shops/helm-charts.git helm-charts

                    helm secrets upgrade --install ${RELEASE_NAME} ${MICROSERVICES_CHART_NAME} -n ${NAMESPACE} -f ${MICROSERVICES_CHART_NAME}values-${NAMESPACE}.yaml -f ${MICROSERVICES_CHART_NAME}secrets.yaml.sops.yaml

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
                            kubectl get all -n $NAMESPACE
                            '''
                        } else if (userChoice == 'Rollback') {
                            echo "Rollback du déploiement..."
                            sh '''
                            helm rollback $RELEASE_NAME 0 --wait --namespace $NAMESPACE || echo "Aucune révision précédente disponible pour rollback de ${RELEASE_NAME}"
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
