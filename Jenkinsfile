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
        NEXUS_URL      = 'http://10.0.141.246:8081'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'rm -rf reports && mkdir -p reports'
            }
        }

        stage('Compile') {
            steps { sh 'mvn -B clean compile' }
        }

        stage('Unit Tests') {
            steps { sh 'mvn -B test' }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    sh 'cat target/surefire-reports/*.txt > reports/unit-test-summary.txt 2>/dev/null || true'
                }
            }
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

        stage('SonarQube Report') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                      curl -s -u "$SONAR_AUTH_TOKEN:" "$SONAR_HOST_URL/api/qualitygates/project_status?projectKey=project-board" -o reports/sonarqube-quality-gate.json
                      curl -s -u "$SONAR_AUTH_TOKEN:" "$SONAR_HOST_URL/api/measures/component?component=project-board&metricKeys=bugs,vulnerabilities,code_smells,security_hotspots,coverage,duplicated_lines_density,ncloc" -o reports/sonarqube-measures.json
                      curl -s -u "$SONAR_AUTH_TOKEN:" "$SONAR_HOST_URL/api/issues/search?componentKeys=project-board&resolved=false&ps=500" -o reports/sonarqube-issues.json
                    '''
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

        stage('Nexus Report') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NX_USER', passwordVariable: 'NX_PASS')]) {
                    sh '''
                      curl -s -u "$NX_USER:$NX_PASS" "$NEXUS_URL/service/rest/v1/search?repository=maven-snapshots&sort=version&direction=desc" -o reports/nexus-artifacts.json
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps { sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .' }
        }

        stage('Trivy Scan') {
            steps {
                // exit-code 0 = report only. Change to 1 to fail the build on HIGH/CRITICAL findings.
                sh '''
                  export TMPDIR=/var/lib/jenkins/trivy/tmp
                  trivy image --cache-dir /var/lib/jenkins/trivy/cache --exit-code 0 --severity HIGH,CRITICAL --no-progress --format table -o reports/trivy-report.txt ${IMAGE_NAME}:${IMAGE_TAG}
                  trivy image --cache-dir /var/lib/jenkins/trivy/cache --exit-code 0 --severity HIGH,CRITICAL --no-progress --format json -o reports/trivy-report.json ${IMAGE_NAME}:${IMAGE_TAG}
                  tail -n 25 reports/trivy-report.txt || true
                '''
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
    }

    post {
        success {
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
            emailext(
                to: 'uaher97.ca@gmail.com',
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} deployed",
                body: "Deployment succeeded.\nImage: ${env.IMAGE_NAME}:${env.IMAGE_TAG}\nBuild: ${env.BUILD_URL}\n\nAttached: SonarQube, Trivy, Nexus and unit test reports, and the build log.",
                attachmentsPattern: 'reports/*',
                attachLog: true,
                compressLog: true
            )
        }
        failure {
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
            emailext(
                to: 'uaher97.ca@gmail.com',
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed.\nBuild: ${env.BUILD_URL}\n\nAttached: the build log and any reports produced before the failure.",
                attachmentsPattern: 'reports/*',
                attachLog: true,
                compressLog: true
            )
        }
        always { sh 'docker logout || true' }
    }
}
