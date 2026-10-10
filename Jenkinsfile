pipeline {
    agent any

    environment {
        DOCKER_USER = 'medrahim7'
        IMAGE_TAG   = "${BUILD_NUMBER}"
    }

    stages {
        // ===================== CI =====================
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh 'mvn -B clean compile'
                }
            }
        }

        stage('Test Backend') {
            steps {
                dir('backend') {
                    sh 'mvn -B test'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                sh '''
                    docker start sonarqube >/dev/null 2>&1 || true
                    for i in $(seq 1 60); do
                        curl -s http://localhost:9000/api/system/status | grep -q '"UP"' && break
                        sleep 10
                    done
                '''
                dir('backend') {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn -B sonar:sonar'
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package Backend') {
            steps {
                dir('backend') {
                    sh 'mvn -B package -DskipTests'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                    sh 'npm run build'
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'backend/target/*.jar, frontend/dist/**', fingerprint: true
            }
        }

        // ===================== CD =====================
        stage('Build Docker Images') {
            steps {
                sh 'docker compose build'
            }
        }

        stage('Push Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                    retry(3) {
                        sh 'docker compose push backend frontend'
                    }
                    retry(3) {
                        sh '''
                            for img in gp-backend gp-frontend; do
                              docker tag $DOCKER_USER/$img:$IMAGE_TAG $DOCKER_USER/$img:latest
                              docker push $DOCKER_USER/$img:latest
                            done
                        '''
                    }
                }
            }
        }

        stage('Deploy (Docker Compose)') {
            steps {
                sh 'docker compose up -d'
                sh 'docker compose ps'
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
    }
}
