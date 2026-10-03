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
                    sh 'mvn -B clean package -DskipTests'
                }
            }
        }

        stage('Test Backend') {
            steps {
                sh '''
                    docker rm -f ci-mysql >/dev/null 2>&1 || true
                    docker run -d --name ci-mysql -p 3307:3306 \
                        -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=test_db mysql:8.0
                    for i in $(seq 1 60); do
                        docker exec ci-mysql mysql -h127.0.0.1 -uroot -proot -e "SELECT 1" >/dev/null 2>&1 && break
                        sleep 3
                    done
                '''
                dir('backend') {
                    withEnv(['SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3307/test_db?createDatabaseIfNotExist=true']) {
                        sh 'mvn -B test'
                    }
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
                    sh 'docker rm -f ci-mysql || true'
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

        // ===================== CD (Docker) =====================
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
