pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    environment {
        DOCKERHUB_USER = 'utka9712'
        IMAGE_NAME     = "${DOCKERHUB_USER}/project-board"
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Compile') {
            steps { sh 'mvn -B clean compile' }
        }

        stage('Unit Tests') {
            steps { sh 'mvn -B test' }
            post { always { junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml' } }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=project-board -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.token=$SONAR_AUTH_TOKEN'
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

        stage('Package (executable file)') {
            steps { sh 'mvn -B package -DskipTests' }
        }

        stage('Publish to Nexus') {
            steps {
                configFileProvider([configFile(fileId: 'maven-settings', variable: 'MAVEN_SETTINGS')]) {
                    sh 'mvn -B -s $MAVEN_SETTINGS deploy -DskipTests'
                }
            }
        }

        stage('Build Docker Image') {
            steps { sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .' }
        }

        stage('Trivy Scan') {
            steps {
                // exit-code 0 = report only. Change to 1 to fail the build on HIGH/CRITICAL findings.
                sh 'trivy image --exit-code 0 --severity HIGH,CRITICAL --no-progress ${IMAGE_NAME}:${IMAGE_TAG}'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                      echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                      docker push ${IMAGE_NAME}:${IMAGE_TAG}
                      docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        /* ---- Enable after the EKS cluster exists ----
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                      sed "s|IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|" k8s/deployment.yaml | kubectl apply -f -
                      kubectl apply -f k8s/service.yaml
                      kubectl rollout status deployment/project-board --timeout=180s
                    '''
                }
            }
        }
        ---- end of Kubernetes stage ---- */
    }

    post {
        success {
            mail to: 'uaher97.ca@gmail.com',
                 subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Pipeline succeeded.\nImage: ${IMAGE_NAME}:${IMAGE_TAG}\nBuild: ${env.BUILD_URL}"
        }
        failure {
            mail to: 'uaher97.ca@gmail.com',
                 subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build failed. Check: ${env.BUILD_URL}"
        }
        always { sh 'docker logout || true' }
    }
}
