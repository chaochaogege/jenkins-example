pipeline {
    agent any
    stages {
        stage('build') {
            steps {
                sh 'mkdir build21'
            }
        }
        stage('test') {
            steps {
                sh 'touch test.txt'
            }
        }
        stage('deploy') {
            steps {
                echo '发布'
            }
        }
    }
}
