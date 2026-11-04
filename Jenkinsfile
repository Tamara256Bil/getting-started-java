pipeline {
    agent { label 'java-jdk21-jenkins-agent-1' }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    parameters {
        string(name: 'BASE_URL', defaultValue: 'https://github.com', description: 'Базовый URL для тестов')
        choice(name: 'BROWSER', choices: ['chrome'], description: 'Браузер для тестов')
        booleanParam(name: 'HEADLESS', defaultValue: true, description: 'Режим headless')
    }
    
    triggers {
        pollSCM('H/5 * * * *')
        cron('H 2 * * *')
    }
    
    environment {
        TZ = 'UTC'
    }
    
    stages {
        stage('Diag: Allure') {
            steps {
                sh 'which allure && allure --version || echo "Allure CLI not found"'
            }
        }
        
        stage('Diag: Chrome') {
            steps {
                sh 'google-chrome --version || chromium --version || chromium-browser --version || echo "Chrome not found"'
                sh 'google-chrome --headless --disable-gpu --no-sandbox --dump-dom https://example.com 2>&1 | head -5 || echo "Chrome headless test failed"'
            }
        }
        
        stage('Diag: Parameters') {
            steps {
                sh 'echo "BASE_URL=${params.BASE_URL}"'
            }
        }
        
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh './gradlew --no-daemon clean assemble'
            }
        }
        
        stage('Tests') {
            steps {
                sh """
                    ./gradlew test --no-daemon \
                        -DbaseUrl="${params.BASE_URL}" \
                        -Dbrowser="${params.BROWSER}" \
                        -Dheadless="${params.HEADLESS}"
                """
            }
            post {
                always {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                        junit testResults: 'build/allure-results/*.xml', allowEmptyResults: false
                    }
                }
            }
        }
        
        stage('Generate Allure Report') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    sh 'allure generate build/allure-results --clean -o build/allure-report'
                }
            }
        }
        
        stage('Publish Allure Report') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    allure([includeProperties: false, results: [[path: 'build/allure-results']]])
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'build/allure-results/**/*', allowEmptyArchive: true
            archiveArtifacts artifacts: 'build/allure-report/**/*', allowEmptyArchive: true
        }
        failure {
            echo 'Build failed. Check logs for details.'
        }
        unstable {
            echo 'Tests failed. Check JUnit report for details.'
        }
    }
}