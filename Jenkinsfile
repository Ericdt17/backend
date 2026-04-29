pipeline {
    agent any

    stages {

        stage('Checkout / Workspace') {
            steps {
                sh 'pwd'
                sh 'ls -la /var/jenkins_home/projects/api_crisiview'
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('/var/jenkins_home/projects/api_crisiview') {
                    sh 'npm install'
                }
            }
        }

        stage('Run Tests (CI FIXED)') {
            steps {
                dir('/var/jenkins_home/projects/api_crisiview') {
                    sh '''
                    export DB_HOST=mysql_exam_compose
                    export DB_PORT=3306
                    export DB_USER=root
                    export DB_PASSWORD=root
                    export DB_NAME=exam

                    # Wait for MySQL to accept connections
                    echo "Waiting for MySQL..."
                    for i in $(seq 1 30); do
                        node -e "
                            import('mysql2/promise').then(m =>
                                m.default.createConnection({
                                    host: 'mysql_exam_compose',
                                    port: 3306,
                                    user: 'root',
                                    password: 'root'
                                })
                            ).then(() => { console.log('MySQL ready'); process.exit(0); })
                             .catch(() => process.exit(1));
                        " && break
                        echo "Attempt $i/30 - waiting for MySQL..."
                        sleep 2
                    done

                    NODE_ENV=test \
                    NODE_OPTIONS=--experimental-vm-modules \
                    npx jest --runInBand --testTimeout=60000
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        dir('/var/jenkins_home/projects/api_crisiview') {
                            sh '''
                            docker run --rm \
                            --network deploy_default \
                            -v $PWD:/usr/src \
                            sonarsource/sonar-scanner-cli \
                            -Dsonar.projectKey=crisisview-backend \
                            -Dsonar.sources=/usr/src \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=${SONAR_AUTH_TOKEN} \
                            -Dsonar.exclusions=node_modules/**,__tests__/** \
                            -Dsonar.sourceEncoding=UTF-8
                            '''
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Check') {
            steps {
                sh 'echo "Docker stage OK"'
            }
        }
    }

    post {
        success {
            echo 'BACKEND PIPELINE SUCCESS'
        }
        failure {
            echo 'BACKEND PIPELINE FAILED'
        }
    }
}
