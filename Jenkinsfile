pipeline {
    environment {
        AWS_REGION = 'us-east-1'
        CLUSTER_NAME = 'sockshop-EKS-VPC'
        CHART_NAME = 'helm-charts/other-microservices/'
        NAMESPACE = 'dev'
        RELEASE_NAME = 'socksshop-microservices'
    }
    agent any

    stages {
        // stage('Dev deployment') {
        //     environment {
        //         KUBECONFIG = credentials("config")
        //     }
        //     steps {
        //         script {
        //             withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
        //                 sh '''
        //                 rm -Rf .kube
        //                 mkdir .kube
        //                 ls
        //                 cat $KUBECONFIG > .kube/config

        //                 rm -Rf helm-charts
        //                 git clone https://$GIT_USER:$GIT_PASS@github.com/socks-shops/helm-charts.git helm-charts

        //                 helm dependency update helm-charts/other-microservices/
        //                 helm upgrade --install socksshop-microservices ./helm-charts/other-microservices/ --namespace dev
        //                 kubectl get all -n dev
        //                 sleep 15
        //                 '''
        //             }
        //         }
        //     }
        // }

        // stage('Test AWS Connection') {
        //     steps {
        //         withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
        //             sh 'aws sts get-caller-identity'
        //         }
        //     }
        // }

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

        // stage('Health check') {
        //     steps {
        //         script {
        //             sh '''
        //             curl -s -o /dev/null -w "%{http_code}" http://localhost:30001 | grep 200
        //             curl http://localhost:30001 | grep "WeaveSocks"
        //             curl -s -o /dev/null -w "%{http_code}" http://socks-shop.local | grep 200
        //             curl http://socks-shop.local | grep "WeaveSocks"
        //             '''
        //         }
        //     }
        // }

        // stage('Déploiement en prod') {
        //     when {
        //         expression { env.GIT_BRANCH ==~ /(origin\/)?main/ }
        //     }
        //     environment {
        //         KUBECONFIG = credentials("config")
        //     }
        //     steps {
        //         timeout(time: 15, unit: "MINUTES") {
        //             input message: 'Do you want to deploy in production ?', ok: 'Yes'
        //         }
        //         script {
        //             sh '''
        //             rm -Rf .kube
        //             mkdir .kube
        //             ls
        //             cat $KUBECONFIG > .kube/config
        //             helm dependency update charts/
        //             helm upgrade --install jenkins-devops ./charts/ --namespace prod
        //             kubectl get all -n prod
        //             '''
        //         }
        //     }
        // }
    }
}
