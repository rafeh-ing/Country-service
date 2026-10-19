pipeline {
    agent any

    tools {
        jdk 'JDK21'
        maven 'mymaven'
    }

    stages {
        stage('Checkout code') {
            steps {
                checkout scm
            }
        }

        stage('Compile, test, package') {
            steps {
                sh 'mvn clean verify'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(installationName: 'MySonarQubeServer', credentialsId: 'sonarqubePWD') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=country-service -Dsonar.projectName=country-service'
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

        stage('Deploy war to Nexus') {
            steps {
                sh 'mvn deploy -DskipTests'
            }
        }

        stage('Publish reports to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-creds',
                                                  usernameVariable: 'NEXUS_USER',
                                                  passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        cd target
                        tar czf jacoco-report.tar.gz site/jacoco
                        tar czf surefire-reports.tar.gz surefire-reports
                        BASE=http://localhost:8081/repository/ci-reports/country-service/${BUILD_NUMBER}
                        curl -sf -u "$NEXUS_USER:$NEXUS_PASS" --upload-file jacoco-report.tar.gz "$BASE/jacoco-report.tar.gz"
                        curl -sf -u "$NEXUS_USER:$NEXUS_PASS" --upload-file surefire-reports.tar.gz "$BASE/surefire-reports.tar.gz"
                    '''
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                sh 'cp target/country-service.war /opt/tomcat-webapps/country-service.war'
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'target/*.war', fingerprint: true
        }
    }
}
