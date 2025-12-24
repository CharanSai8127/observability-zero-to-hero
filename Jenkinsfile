pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_REGISTRY      = 'charansait372'
        IMAGE_TAG            = 'v1.2'

        IMAGE_NAME_A = "${DOCKER_REGISTRY}/observability-service-a:${IMAGE_TAG}"
        IMAGE_NAME_B = "${DOCKER_REGISTRY}/observability-service-b:${IMAGE_TAG}"

        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/CharanSai8127/observability-zero-to-hero.git'
            }
        }

        stage('Run Node.js Tests') {
            steps {
                script {
                    dir('day-4/application/service-a') {
                        sh '''
                          npm install
                          npm test || true
                          npm audit || true
                        '''
                    }

                    dir('day-4/application/service-b') {
                        sh '''
                          npm install
                          npm test || true
                          npm audit || true
                        '''
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    dir('day-4/application/service-a') {
                        sh "docker build -t ${IMAGE_NAME_A} ."
                    }

                    dir('day-4/application/service-b') {
                        sh "docker build -t ${IMAGE_NAME_B} ."
                    }
                }
            }
        }

        stage('Trivy Image Scan (Reports)') {
            steps {
                script {
                    sh '''
                      mkdir -p reports/service-a reports/service-b

                      trivy image --severity HIGH,CRITICAL \
                        --format json \
                        --output reports/service-a/trivy.json \
                        ${IMAGE_NAME_A} || true

                      trivy image --severity HIGH,CRITICAL \
                        --format json \
                        --output reports/service-b/trivy.json \
                        ${IMAGE_NAME_B} || true
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    script {

                        dir('day-4/application/service-a') {
                            sh '''
                              $SCANNER_HOME/bin/sonar-scanner \
                                -Dsonar.projectKey=observability-service-a \
                                -Dsonar.sources=.
                              mkdir -p ../../../reports/service-a/sonar
                              cp .scannerwork/report-task.txt ../../../reports/service-a/sonar/ || true
                            '''
                        }

                        dir('day-4/application/service-b') {
                            sh '''
                              $SCANNER_HOME/bin/sonar-scanner \
                                -Dsonar.projectKey=observability-service-b \
                                -Dsonar.sources=.
                              mkdir -p ../../../reports/service-b/sonar
                              cp .scannerwork/report-task.txt ../../../reports/service-b/sonar/ || true
                            '''
                        }
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'owasp-cred',
                        usernameVariable: 'OSSINDEX_USER',
                        passwordVariable: 'OSSINDEX_TOKEN'
                    )
                ]) {
                    script {

                        dir('day-4/application/service-a') {
                            dependencyCheck(
                                additionalArguments: '--scan .',
                                odcInstallation: 'owasp'
                            )
                        }

                        dir('day-4/application/service-b') {
                            dependencyCheck(
                                additionalArguments: '--scan .',
                                odcInstallation: 'owasp'
                            )
                        }

                        dependencyCheckPublisher(
                            pattern: '**/dependency-check-report.xml'
                        )
                    }
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push ${IMAGE_NAME_A}
                      docker push ${IMAGE_NAME_B}
                      docker logout
                    '''
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'reports/**', fingerprint: true
            echo 'CI completed successfully. All Sonar, Trivy, and OWASP reports archived.'
        }

        failure {
            echo 'CI failed. Review logs and archived artifacts.'
        }
    }
}

