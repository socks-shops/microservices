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
                    def currentBranch = env.GIT_BRANCH // Jenkins variable

                    if (currentBranch == 'origin/main') {
                        env.NAMESPACE = 'dev'
                    } else if (currentBranch == 'origin/staging') {
                        env.NAMESPACE = 'staging'
                    } else if (currentBranch == 'origin/prod') {
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

                    # --- DÉBUT DES ÉTAPES DE DIAGNOSTIC ---

                    echo "--- Diagnostic de l'environnement ---"

                    # 3. Vérifier que SOPS_AGE_KEY est bien présente (mais pas sa valeur pour la sécurité)
                    if [ -n "$SOPS_AGE_KEY" ]; then
                        echo "SOPS_AGE_KEY est définie."
                    else
                        echo "ERREUR: SOPS_AGE_KEY n'est PAS définie ou est vide !!"
                        exit 1 # Arrêter si la clé n'est pas là
                    fi

                    # 4. Vérifier la présence et l'exécutabilité des outils
                    echo "Vérification des binaires sops, age, age-keygen..."
                    which sops || echo "ERREUR: sops non trouvé dans le PATH!"
                    which age || echo "ERREUR: age non trouvé dans le PATH!"
                    which age-keygen || echo "ERREUR: age-keygen non trouvé dans le PATH!"

                    # 5. Vérifier l'installation du plugin helm-secrets
                    echo "Vérification du plugin helm-secrets..."
                    helm plugin list || echo "ERREUR: helm plugin list a échoué. Le plugin helm-secrets n'est peut-être pas installé ou détecté."
                    helm secrets --help > /dev/null 2>&1 || echo "ERREUR: La commande 'helm secrets' n'est pas reconnue!"

                    # 6. Tenter de déchiffrer le secret manuellement (TRÈS IMPORTANT !)
                    echo "Tentative de déchiffrement manuel du secret avec sops..."
                    # Le chemin de votre secret dans le dépôt cloné
                    ENCRYPTED_SECRET_FILE="${MICROSERVICES_CHART_NAME}templates/catalogue-db-secret.yaml.sops.yaml"

                    # Utilisation d'un temporaire pour ne pas exposer le secret en clair dans les logs
                    if sops --decrypt "$ENCRYPTED_SECRET_FILE" > /tmp/decrypted_secret.yaml; then
                        echo "Déchiffrement manuel réussi !"
                        cat /tmp/decrypted_secret.yaml # Affiche le contenu déchiffré (peut être sensible, attention aux logs)
                        rm /tmp/decrypted_secret.yaml # Nettoyer le fichier temporaire
                    else
                        echo "ERREUR: Le déchiffrement manuel de $ENCRYPTED_SECRET_FILE a ÉCHOUÉ !"
                        echo "Vérifiez : 1. La clé privée SOPS_AGE_KEY. 2. Si le secret est bien chiffré avec cette clé."
                        exit 1 # Arrêter si le déchiffrement manuel échoue
                    fi

                    echo "--- Fin du Diagnostic ---"

                    helm secrets upgrade --install ${RELEASE_NAME} ${MICROSERVICES_CHART_NAME} -n ${NAMESPACE}

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
