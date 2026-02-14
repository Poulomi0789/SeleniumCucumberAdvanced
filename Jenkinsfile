pipeline {
    agent {
        docker { 
            image 'markhobson/maven-chrome:jdk-17' 
            args '-u root --shm-size=2g'
        }
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['qa', 'dev', 'prod'], description: 'Select properties file')
        string(name: 'THREADS', defaultValue: '2', description: 'Parallel browser count')
        choice(name: 'TAGS', choices: ['@smoke', '@regression', '@all'], description: 'Cucumber tags')
    }

    environment {
        ALLURE_RESULTS = 'target/allure-results'
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
                        // Added -Dcucumber.filter.tags to use your TAGS parameter
                        sh "mvn clean test -Denv=${params.ENVIRONMENT} -Ddataproviderthreadcount=${params.THREADS} -Dcucumber.filter.tags=${params.TAGS}"
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
                    }
                }
            }
        }

        stage('Publish Results') {
            steps {
                allure includeProperties: false, results: [[path: 'target/allure-results']]
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
                subject: "✅ SUCCESS: UI Tests ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """<h2>Build Successful 🚀</h2>
                         <b>Environment:</b> ${params.ENVIRONMENT} <br>
                         <b>Tags Run:</b> ${params.TAGS} <br>
                         <b>Allure Report:</b> <a href="${env.BUILD_URL}allure">View Online</a>""",
                attachmentsPattern: 'allure-report.zip',
                to: "${env.EMAIL_RECIPIENTS}"
            )
        }

        unstable {
            emailext(
                subject: "⚠️ UNSTABLE: UI Tests ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """<h2>Tests Failed ⚠️</h2>
                         <b>Environment:</b> ${params.ENVIRONMENT} <br>
                         <b>Check Allure:</b> <a href="${env.BUILD_URL}allure">View Report</a>""",
                attachmentsPattern: 'allure-report.zip',
                to: "${env.EMAIL_RECIPIENTS}"
            )
        }

        failure {
            emailext(
                subject: "❌ FAILURE: UI Test ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """<h2>Pipeline Error ❌</h2>
                         <b>Build crashed.</b><br>
                         <a href="${env.BUILD_URL}console">Console Output</a>""",
                to: "${env.EMAIL_RECIPIENTS}"
            )
        }
    }
}
