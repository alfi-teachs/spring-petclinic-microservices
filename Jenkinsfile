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
                echo '>>> Step 1: Getting code from GitHub...'
                checkout scm
                echo '>>> Checkout finished ✅'
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
        stage ('SonarQube Analysis') {
            steps {
                echo '>>> Step 4: Sending code to SonarCloud for quality check...'
                withSonarQubeEnv('sonarcloud') {
                    sh '''./mvnw -B sonar:sonar \
                        -Dsonar.organization=alfi-teachs \
                        -Dsonar.projectKey=alfi-teachs_spring-petclinic-microservices'''
                }
                echo '>>> SonarQube analysis finished ✅'
            }
        }
        stage('Package') {
            steps {
                echo '>>> Step 5:  Packaging apps into .jar files...'
                sh './mvnw -B package -DskipTests'
                sh 'ls -lh spring-petclinic-*/target/*.jar'
                echo '>>> Package finished ✅'
            }
        }
    }
}