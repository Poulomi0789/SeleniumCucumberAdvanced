pipeline {
    agent {
        docker {
            image 'markhobson/maven-chrome:jdk-17'
            args '-u root --shm-size=2g'
        }
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['qa', 'dev', 'prod'],
            description: 'Select which environment properties file to use'
        )

        string(
            name: 'THREADS',
            defaultValue: '2',
            description: 'How many browser instances to run at once'
        )

        choice(
            name: 'TAGS',
            choices: ['@smoke', '@regression', '@all'],
            description: 'Filter which scenarios to run'
        )
    }

    environment {
        ALLURE_RESULTS = 'target/allure-results'
        GIT_URL = 'https://github.com/Poulomi0789/SeleniumCucumberAdvanced.git'
        GIT_BRANCH = 'main'
        EMAIL_RECIPIENTS = 'poulomidas89@gmail.com'
    }

    stages {

        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('UI Automation Execution') {
            steps {
                script {
                    try {
                        sh "mvn clean test -Denv=${params.ENVIRONMENT} -Ddataproviderthreadcount=${params.THREADS}"
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        echo "Tests failed, proceeding to report generation..."
                    }
                }
            }
        }

        stage('Generate & Zip Report') {
            steps {
                sh 'mvn io.qameta.allure:allure-maven:report'

                script {
                    if (fileExists('target/site/allure-maven-plugin')) {
                        zip zipFile: 'allure-report.zip',
                            dir: 'target/site/allure-maven-plugin',
                            archive: true
                    } else {
                        echo "Warning: Allure report directory not found, skipping zip."
                    }
                }
            }
        }

        stage('Publish to Jenkins') {
            steps {
                allure includeProperties: false, results: [
                    [path: 'target/allure-results']
                ]
            }
        }
    }

    post {

        always {
            archiveArtifacts artifacts: '**/*.log', allowEmptyArchive: true
            junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
        }

        success {
            emailext(
                subject: "✅ Selenium Cucumber Tests Passed | ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """<h2>Build Successful 🚀</h2>
                         <b>Environment:</b> ${params.ENVIRONMENT} <br>
                         <b>Tags Run:</b> ${params.TAGS} <br>
                         <b>Allure Report:</b> <a href="${env.BUILD_URL}allure">View Online</a>""",
                attachmentsPattern: 'allure-report.zip',
                mimeType: 'text/html',
                to: "${EMAIL_RECIPIENTS}"
            )
        }

        unstable {
            emailext(
                subject: "⚠️ Selenium Cucumber Tests Unstable | ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """<h2>Tests Failed ⚠️</h2>
                         <b>Environment:</b> ${params.ENVIRONMENT} <br>
                         <b>Check Allure for details:</b> 
                         <a href="${env.BUILD_URL}allure">View Report</a>""",
                attachmentsPattern: 'allure-report.zip',
                mimeType: 'text/html',
                to: "${EMAIL_RECIPIENTS}"
            )
        }

        failure {
            emailext(
                subject: "❌ Selenium Cucumber Tests Failed | ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """<h2>Pipeline Error ❌</h2>
                         <b>The build crashed before finishing.</b><br>
                         <a href="${env.BUILD_URL}console">Console Output</a>""",
                to: "${EMAIL_RECIPIENTS}"
            )
        }
    }
}
