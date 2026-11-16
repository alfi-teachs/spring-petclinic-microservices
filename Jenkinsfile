pipeline {
    agent {label 'docker'}
    options {
        skipDefaultCheckout()
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('compile'){
            steps {
                 echo '>>> Step 2: Compiling Java code with Maven...'
                 sh './mvnw -B clean compile'
                 echo '>>> Compile finished ✅'
            }
        }
        stage('test'){
            steps {
                 echo '>>> Step 3: Running unit tests with Maven...'
                 sh './mvnw -B test'
                 echo '>>> Test finished ✅'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

    }
}