pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    triggers {
        pollSCM('* * * * *') // Poll every minute
    }
    environment {
        BUILD_USER = bat(script: "echo %USERNAME%", returnStdout: true).trim().replaceAll('(?s)^.*?\r?\n', '')
    }
    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/himanshu110796/Jenkins-test'
                script {
                    env.GIT_AUTHOR = bat(script: 'git log -1 --pretty=format:"%%an"', returnStdout: true).trim().replaceAll('(?s)^.*?\r?\n', '')
                    env.GIT_COMMITTER_USERNAME = bat(script: 'git log -1 --pretty=format:"%%ae"', returnStdout: true).trim().replaceAll('(?s)^.*?\r?\n', '')

                    withCredentials([string(credentialsId: 'github-pat', variable: 'GITHUB_PAT')]) {
                        def jsonFile = "git_emails.json"

                        // Fetch GitHub user email and store it in a file
                        bat(script: """
                            curl -s -H "Authorization: token %GITHUB_PAT%" ^
                            "https://api.github.com/user/emails" ^
                            --compressed --max-time 10 > ${jsonFile}
                        """)

                        script {
                            // Read JSON from file and parse it
                            def email = powershell(script: """
                                try {
                                    \$jsonContent = Get-Content ${jsonFile} -Raw
                                    \$data = \$jsonContent | ConvertFrom-Json
                                    \$primaryEmail = (\$data | Where-Object { \$_.primary -eq \$true }).email
                                    if (-not \$primaryEmail) { \$primaryEmail = "no-email-found@example.com" }
                                    Write-Output \$primaryEmail
                                } catch {
                                    Write-Output "no-email-found@example.com"
                                }
                            """, returnStdout: true).trim()

                            env.GIT_COMMITTER_EMAIL = email
                        }
                    }
                }
            }
        }
        stage('Build with Maven') {
            steps {
                bat 'mvn clean package'
            }
        }
        stage('Deploy to Tomcat') {
            steps {
                deploy adapters: [tomcat9(
                    credentialsId: 'afac12b6-3a3f-46a5-8338-7645ea36aee2',
                    path: '',
                    url: 'http://localhost:8083/'
                )], contextPath: 'javawebappp', war: 'target/*.war'
            }
        }
    }
    post {
        success {
            script {
                if (!env.GIT_COMMITTER_EMAIL || env.GIT_COMMITTER_EMAIL == "no-email-found@example.com") {
                    env.GIT_COMMITTER_EMAIL = "fallback@example.com"
                }
            }
            emailext (
                subject: "✅ SUCCESS: Jenkins Build - ${JOB_NAME} #${BUILD_NUMBER}",
                body: """<html>
                    <body>
                        <h3 style="color:green;">Good news! ✅</h3>
                        <p>The Jenkins build for <b>${JOB_NAME}</b> (#${BUILD_NUMBER}) was successful.</p>
                        <p><b>Triggered by:</b> ${BUILD_USER}</p>
                        <p><b>Latest Commit:</b> ${GIT_AUTHOR}</p>
                        <p><b>Commit Email:</b> ${GIT_COMMITTER_EMAIL}</p>
                        <p>Check the build details here: <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                    </body>
                </html>""",
                to: "${GIT_COMMITTER_EMAIL}",
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html'
            )
        }
        failure {
            script {
                if (!env.GIT_COMMITTER_EMAIL || env.GIT_COMMITTER_EMAIL == "no-email-found@example.com") {
                    env.GIT_COMMITTER_EMAIL = "fallback@example.com"
                }
            }
            emailext (
                subject: "❌ FAILURE: Jenkins Build - ${JOB_NAME} #${BUILD_NUMBER}",
                body: """<html>
                    <body>
                        <h3 style="color:red;">Build Failed ❌</h3>
                        <p>The Jenkins build for <b>${JOB_NAME}</b> (#${BUILD_NUMBER}) has failed.</p>
                        <p><b>Triggered by:</b> ${BUILD_USER}</p>
                        <p><b>Latest Commit:</b> ${GIT_AUTHOR}</p>
                        <p><b>Commit Email:</b> ${GIT_COMMITTER_EMAIL}</p>
                        <p>Please check the logs: <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                    </body>
                </html>""",
                to: "${GIT_COMMITTER_EMAIL}",
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html'
            )
        }
    }
}
