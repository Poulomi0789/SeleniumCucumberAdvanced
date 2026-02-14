pipeline {
    agent {
        docker { 
            image 'markhobson/maven-chrome:jdk-17' 
            args '-u root --shm-size=2g'
        }
    }

    parameters {
        // 1. Environment Selection
        choice(
            name: 'ENVIRONMENT', 
            choices: ['qa', 'dev', 'prod'], 
            description: 'Select which environment properties file to use (qa.properties vs dev.properties)'
        )
        
        // 2. Parallel Thread Count
        string(
            name: 'THREADS', 
            defaultValue: '2', 
            description: 'How many browser instances to run at once'
        )

        // 3. Optional: Test Tag Selection (Smoke vs Regression)
        choice(
            name: 'TAGS', 
            choices: ['@smoke', '@regression', '@all'], 
            description: 'Filter which scenarios to run'
        )
    }
    environment {
        ALLURE_RESULTS = 'target/allure-results'
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
                        // -Denv: Switched based on Jenkins parameter
                        // -Ddataproviderthreadcount: Controls Cucumber parallel execution
                        sh "mvn clean test -Denv=${params.ENVIRONMENT} -Ddataproviderthreadcount=${params.THREADS}"
                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        echo "Tests failed, proceeding to report generation..."
                    }
                }
            }
        }
    }

    post {
        always {
            // Generate Allure Report
            script {
                allure includeProperties: false,
                       jdk: '',
                       results: [[path: "${env.ALLURE_RESULTS}"]]
            }
        }

        failure {
            emailext body: """
                <h3>UI Automation Build Failed</h3>
                <p>Build: ${env.BUILD_URL}</p>
                <p>Check the attached Allure Report for screenshots of failed steps.</p>
            """,
            subject: "ALERT: UI Test Failure - ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
            to: 'qa-team@company.com'
        }

        success {
            emailext body: "UI Automation passed successfully. Build: ${env.BUILD_URL}",
                     subject: "SUCCESS: UI Test Passed - ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                     to: 'qa-team@company.com'
        }
    }
}
