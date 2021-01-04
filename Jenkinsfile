pipeline {

    agent none
    
    triggers { pollSCM('*/1 * * * *') }
    
    environment {
        USER = "xxx"
        MAIL = "xxxxxxxxxx@xxx.com"
        PROJECT = "GAME"
        PREPARATION = "None"
        BUILD = "None"
        DEPLOY = "None"
        TEST = "None"
    }

    stages {
        stage('Preparation') {
            agent {
                label 'node1'
            }
            steps {
                git branch: 'xxxxxxx', url: 'https://xxxxxxxxxxxxx.git'
            }
            post {
                always {
                    echo "The code pull action is completed"
                }
                failure {
                    script {
                        echo "Failed to execute code pull action!!!"
                        PREPARATION = "FAILED"
                    }
                }
                success {
                    script {
                        echo "Successfully execute code pull action!!!"
                        PREPARATION = "SUCCESS"
                    }
                }
            }
        }
        stage('Build') {
            agent {
                label 'node1'
            }
            steps {
                echo "gradle build will go here."
            }
            post {
                always {
                    echo "Build stage complete"
                }
                failure {
                    script {
                        echo "Build failed!!!"
                        BUILD = "FAILED"
                    }
                }
                success {
                    script {
                        echo "Build succeeded!!!"
                        BUILD = "SUCCESS"
                    }
                }
            }
        }
        stage('Deploy') {
            agent {
                label 'node1'
            }
            steps {
                echo "Deploy the project environment."
            }
        }
        stage('Test') {
            parallel {
                stage ('set1') {
                    agent {
                        label 'node1'
                    }
                    steps {
                        echo "Test project environment. node1"
                        echo "Test project environment. node2"
                        echo "Test project environment. node3"
                    }                    
                }
                stage ('set2') {
                    agent {
                        label 'master'
                    }
                    steps {
                        echo "Test project environment. master1"
                        echo "Test project environment. master2"
                        echo "Test project environment. master3"
                    }                    
                }                
            }

        }
    }
    post {
        always {
            echo "All executed"
        }
        failure {
            script {
                echo "Unfortunately failed!!!"
                mail to: "${MAIL}",
                subject: "项目:${PROJECT}发布失败",
                body: "Preparation=${PREPARATION}\nBUILD=${BUILD}\nDEPLOY=${DEPLOY}\nTEST=${TEST}"
            }
        }
        success {
            script {
                echo "Great success!!!"
                mail to: "${MAIL}",
                subject: "项目:${PROJECT}发布成功",
                body: "Preparation=${PREPARATION}\nBUILD=${BUILD}\nDEPLOY=${DEPLOY}\nTEST=${TEST}"
            }
        }
    }
}
