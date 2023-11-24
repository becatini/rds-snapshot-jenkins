pipeline {
    agent any
    
    environment {
     var1 = ""
    }
    
    stages {
        stage('1') {
            steps {
                script {
                    var1="1"
                    echo var1
                }
            }
        }
        
        stage('2-sh') {
            steps {
                script {
                    var1="2"
                    sh "echo ${var1}"
                }
            }
        }
    }
}
