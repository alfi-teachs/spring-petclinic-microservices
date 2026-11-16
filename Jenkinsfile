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
                './mvnw -B clean compile'
            }
        }

    }
}