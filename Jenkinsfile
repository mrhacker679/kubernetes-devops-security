pipeline {
  agent any

  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later (removed)
            }
        }   
// this stage has been added to test 
      stage('Unit Tests') {
          steps {
            sh "mvn test"
          }
      }   
    }
}
