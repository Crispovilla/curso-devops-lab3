pipeline {
    agent any

    environment {
        SEM_VER        = '1.0.0'
        DOCKERHUB_REPO = 'cvillarroel/curso-devops-lab3'
        GITHUB_REPO    = 'ghcr.io/crispovilla/curso-devops-lab3'
        K8S_NAMESPACE  = 'cvillarroel'
    }

    stages {
        stage('a. Instalación de dependencias') {
            agent {
                docker { image 'node:20' }
            }
            steps {
                sh 'npm ci'
            }
        }

        stage('b. Ejecución de pruebas') {
            agent {
                docker { image 'node:20' }
            }
            steps {
                sh 'npm test'
            }
        }

        stage('c. Cobertura SonarQube & Quality Gate') {
            agent {
                docker { image 'node:20' }
            }
            steps {
                sh 'npm run test:cov'
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                      npx sonarqube-scanner \
                        -Dsonar.host.url=http://host.docker.internal:8082 \
                        -Dsonar.token=${SONAR_TOKEN} \
                        -Dsonar.projectKey=curso-devops-lab3 \
                        -Dsonar.sources=src \
                        -Dsonar.tests=src \
                        -Dsonar.test.inclusions=**/*.spec.ts \
                        -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                        -Dsonar.typescript.lcov.reportPaths=coverage/lcov.info
                    '''
                }
            }
        }

        stage('d. Build de la aplicación') {
            agent {
                docker { image 'node:20' }
            }
            steps {
                sh 'npm run build'
            }
        }

        stage('e. Construcción imagen Docker multistage') {
            steps {
                sh "docker build -t ${DOCKERHUB_REPO}:${BUILD_NUMBER} ."
            }
        }

        stage('f. Upload a Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                      echo "$PASS" | docker login -u "$USER" --password-stdin
                      docker tag ${DOCKERHUB_REPO}:${BUILD_NUMBER} ${DOCKERHUB_REPO}:${SEM_VER}
                      docker tag ${DOCKERHUB_REPO}:${BUILD_NUMBER} ${DOCKERHUB_REPO}:latest
                      
                      docker push ${DOCKERHUB_REPO}:${BUILD_NUMBER}
                      docker push ${DOCKERHUB_REPO}:${SEM_VER}
                      docker push ${DOCKERHUB_REPO}:latest
                    """
                }
            }
        }

        stage('g. Upload a GitHub Packages') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-packages-credentials', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                      echo "$PASS" | docker login ghcr.io -u "$USER" --password-stdin
                      
                      docker tag ${DOCKERHUB_REPO}:${BUILD_NUMBER} ${GITHUB_REPO}:${BUILD_NUMBER}
                      docker tag ${DOCKERHUB_REPO}:${BUILD_NUMBER} ${GITHUB_REPO}:${SEM_VER}
                      docker tag ${DOCKERHUB_REPO}:${BUILD_NUMBER} ${GITHUB_REPO}:latest
                      
                      docker push ${GITHUB_REPO}:${BUILD_NUMBER}
                      docker push ${GITHUB_REPO}:${SEM_VER}
                      docker push ${GITHUB_REPO}:latest
                    """
                }
            }
        }

        stage('h. Actualización Kubernetes Local') {
            agent {
                docker { 
                    image 'bitnami/kubectl:latest'
                    args '-v /root/.kube:/opt/bitnami/kubectl/.kube:ro --network host'
                }
            }
            steps {
                sh """
                  kubectl set image deployment/app-deployment app-container=${GITHUB_REPO}:${BUILD_NUMBER} -n ${K8S_NAMESPACE}
                  kubectl rollout status deployment/app-deployment -n ${K8S_NAMESPACE} --timeout=60s
                """
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh 'docker logout ghcr.io || true'
        }
    }
}