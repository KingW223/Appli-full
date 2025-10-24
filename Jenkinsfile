pipeline {
    agent any

    tools {
        nodejs "NodeJS_22"
    }

    environment {
        DOCKER_HUB_USER = 'kingwest1'
        FRONT_IMAGE = 'react-frontend'
        BACK_IMAGE  = 'express-backend'
        KUBECONFIG = "C:/Users/HP/.kube/config"
    }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'ref', value: '$.ref'],
                [key: 'pusher_name', value: '$.pusher.name'],
                [key: 'commit_message', value: '$.head_commit.message']
            ],
            causeString: 'Push GitHub par $pusher_name: $commit_message',
            token: 'mysecret',
            printContributedVariables: true,
            printPostContent: true,
            regexpFilterText: '$ref',
            regexpFilterExpression: 'refs/heads/main'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/KingW223/Appli-full.git'
            }
        }

        stage('Install dependencies - Backend') {
            steps {
                dir('back-end') {
                    bat 'npm install'
                }
            }
        }

        stage('Install dependencies - Frontend') {
            steps {
                dir('front-end') {
                    bat 'npm install'
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir('back-end') {
                    bat 'npm test || echo "Aucun test backend ou échec ignoré"'
                }
                dir('front-end') {
                    bat 'npm test || echo "Aucun test frontend ou échec ignoré"'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    bat "docker build -t ${env.DOCKER_HUB_USER}/${env.FRONT_IMAGE}:latest ./front-end"
                    bat "docker build -t ${env.DOCKER_HUB_USER}/${env.BACK_IMAGE}:latest ./back-end"
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'king-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        bat "echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin"
                        bat "docker push ${env.DOCKER_HUB_USER}/${env.FRONT_IMAGE}:latest"
                        bat "docker push ${env.DOCKER_HUB_USER}/${env.BACK_IMAGE}:latest"
                    }
                }
            }
        }

    stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "🚀 Déploiement MongoDB..."
                    bat 'kubectl apply -f k8s/mongodb-deployment.yaml'
    
                    echo "⏳ Attente du démarrage de MongoDB..."
                    // bat 'timeout /t 60 /nobreak'
    
                    echo "🚀 Déploiement Backend..."
                    bat 'kubectl apply -f k8s/backend-deployment.yaml'
                    bat 'kubectl apply -f k8s/backend-service.yaml'
                    // bat 'timeout /t 20 /nobreak'
    
                    echo "🚀 Déploiement Frontend..."
                    bat 'kubectl apply -f k8s/frontend-deployment.yaml'
                    bat 'kubectl apply -f k8s/frontend-service.yaml'
    
                    echo "⏳ Attente des déploiements..."
                    bat '''
                        kubectl rollout status deployment/backend-deployment --timeout=300s
                        kubectl rollout status deployment/frontend-deployment --timeout=300s
                    '''
                }
            }
        }

        stage('Update Kubernetes Images') {
            steps {
                script {
                    bat "kubectl set image deployment/backend-deployment backend=${env.DOCKER_HUB_USER}/${env.BACK_IMAGE}:latest"
                    bat "kubectl set image deployment/frontend-deployment frontend=${env.DOCKER_HUB_USER}/${env.FRONT_IMAGE}:latest"

                    bat '''
                        kubectl rollout status deployment/backend-deployment --timeout=300s
                        kubectl rollout status deployment/frontend-deployment --timeout=300s
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline terminé - vérifiez les logs pour les détails'
            script {
                if (currentBuild.result == 'FAILURE') {
                    bat '''
                        echo "=== Backend Pods ==="
                        kubectl get pods -l app=backend
                        echo "=== Frontend Pods ==="
                        kubectl get pods -l app=frontend
                        echo "=== MongoDB Pods ==="
                        kubectl get pods -l app=mongodb
                        echo "=== Services ==="
                        kubectl get services
                    '''
                }
            }
        }

        success {
            script {
                echo "🎉 DÉPLOIEMENT RÉUSSI !"
                bat '''
                    echo Frontend URL:
                    kubectl get service frontend-service -o jsonpath="{.spec.clusterIP}:{.spec.ports[0].port}"
                    echo Backend URL:
                    kubectl get service backend-service -o jsonpath="{.spec.clusterIP}:{.spec.ports[0].port}"
                '''
                emailext(
                    subject: "SUCCÈS Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: "Le pipeline a réussi ! Consultez : ${env.BUILD_URL}",
                    to: "naziftelecom2@gmail.com"
                )
            }
        }


        failure {
            echo "❌ Le déploiement a échoué."
            emailext(
                subject: "ÉCHEC Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Le pipeline a échoué. Détails : ${env.BUILD_URL}",
                to: "naziftelecom2@gmail.com"
            )
        }

        cleanup {
            bat 'docker logout'
        }
    }
}
