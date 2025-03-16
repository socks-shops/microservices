pipeline {
    agent any

    stages {
        stage('Dev deployment') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh '''
                        rm -Rf .kube
                        mkdir .kube
                        ls
                        cat $KUBECONFIG > .kube/config

                        rm -Rf helm-charts
                        git clone https://$GIT_USER:$GIT_PASS@github.com/socks-shops/helm-charts.git helm-charts
                        ls -R helm-charts/

                        helm dependency update helm-charts/
                        helm upgrade --install socksshop-microservices ./helm-charts/ --namespace dev
                        kubectl get all -n dev
                        sleep 15
                        '''
                    }
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
