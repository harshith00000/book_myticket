pipeline {
    agent any

    environment {
        FAILED_STAGE = "Not Started"
    }

    stages {

        stage('Build') {
            steps {
                script {
                    env.FAILED_STAGE = "Build"

                    sh '''
                    mvn clean package -DskipTests
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    env.FAILED_STAGE = "Deploy"

                    sh '''
                    pkill -f "book-my-ticket-springboot-thymeleaf-master--2-" || true

                    JAR_FILE=$(find target -name "*.jar" | head -1)

                    if [ -z "$JAR_FILE" ]; then
                      echo "No jar file found"
                      exit 1
                    fi

                    nohup java -jar "$JAR_FILE" > app.log 2>&1 &

                    sleep 20
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    env.FAILED_STAGE = "Health Check"

                    sh '''
                    curl -f http://localhost:8080
                    '''
                }
            }
        }
    }

    post {

        success {

            withCredentials([
                string(credentialsId: 'TelegramToken', variable: 'TOKEN'),
                string(credentialsId: 'TelegramID', variable: 'CHAT_ID')
            ]) {

                sh '''
                curl -s -X POST \
                "https://api.telegram.org/bot${TOKEN}/sendMessage" \
                -d chat_id="${CHAT_ID}" \
                -d text="✅ DEPLOYMENT SUCCESS

Project: book-my-ticket-springboot-thymeleaf-master--2-
Build: #${BUILD_NUMBER}
Node: ${NODE_NAME}
Status: Application is running successfully."
                '''
            }
        }

        failure {

            script {

                def appLogs = ""

                try {
                    appLogs = sh(
                        script: "tail -50 app.log 2>/dev/null || echo 'app.log not found'",
                        returnStdout: true
                    ).trim()
                } catch (Exception e) {
                    appLogs = "Unable to read application logs"
                }

                writeFile(
                    file: 'failure_report.txt',
                    text: """
DEPLOYMENT FAILED

Project : book-my-ticket-springboot-thymeleaf-master--2-
Build   : #${env.BUILD_NUMBER}
Stage   : ${env.FAILED_STAGE}

Application Logs:

${appLogs}
"""
                )

                withCredentials([
                    string(credentialsId: 'TelegramToken', variable: 'TOKEN'),
                    string(credentialsId: 'TelegramID', variable: 'CHAT_ID')
                ]) {

                    sh '''
                    curl -s -X POST \
                    "https://api.telegram.org/bot${TOKEN}/sendMessage" \
                    -d chat_id="${CHAT_ID}" \
                    -d text="❌ Deployment Failed

Project: book-my-ticket-springboot-thymeleaf-master--2-
Build: #${BUILD_NUMBER}
Stage: ${FAILED_STAGE}

Detailed logs attached."
                    '''

                    sh '''
                    curl -s -X POST \
                    "https://api.telegram.org/bot${TOKEN}/sendDocument" \
                    -F chat_id="${CHAT_ID}" \
                    -F document=@failure_report.txt
                    '''
                }
            }
        }

        always {
            archiveArtifacts artifacts: 'app.log', allowEmptyArchive: true
        }
    }
}
