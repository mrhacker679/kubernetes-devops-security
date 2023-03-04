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
// this is added as a part of jacoco test          
          post {
            always {
              junit 'target/surefire-reports/*.xml'
              jacoco execPattern: 'target/jacoco.exec'
            }
          }
      }   
// added as a part of docker image build and push
      stage('Docker Build and Push') {
        steps {
          withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
            sh 'printenv'
            sh 'docker build -t shahabuddin7/numericapp:""$GIT_COMMIT"" .'
            sh 'docker push shahabuddin7/numericapp:""$GIT_COMMIT""'
          }
        }
    }
// added as a part of first k8s deployment
      stage('Kubernetes Deployment - Dev') {
        steps {
          withKubeConfig([credentialsId: 'kubeconfig']){
            sh "sed -i 's#replace#shahabuddin7/numericapp:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
            sh "kubectl apply -f k8s_deployment_service.yaml"
        }
        }
    }
}
}
