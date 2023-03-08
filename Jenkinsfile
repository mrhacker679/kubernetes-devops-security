pipeline {
  agent any

    environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "shahabuddin7/numericapp:${GIT_COMMIT}"
    applicationURL="http://139.144.18.62:30512"
    applicationURI="/increment/99"
  }


  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later (removed)
            }
        }   
// this stage has been added to test 
      stage('Unit Tests - JUnit and JaCoCo') {
          steps {
            sh "mvn test"
          }
// this is added as a part of jacoco test          
          // post {
          //   always {
          //     junit 'target/surefire-reports/*.xml'
          //     jacoco execPattern: 'target/jacoco.exec'
          //   }
          // }
      }   

// addded as a part of mutation test from devsecops
    stage('PIT Mutation Test') {
      steps {
        sh "mvn org.pitest:pitest-maven:mutationCoverage"
      }
      // post{
      //   always {
      //     pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
      //   }
      // }
    }

// added as a part of the Sonarqube static code analysis
    stage('SonarQube - SAST') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh "mvn sonar:sonar \
		              -Dsonar.projectKey=numeric-application \
		              -Dsonar.host.url=http://45.79.252.108:9000"
        }
        timeout(time: 2, unit: 'MINUTES') {
          script {
            waitForQualityGate abortPipeline: true
          }
        }
      }
    }

// added as a part of dependency check maven
	stage('Vulnerability Scan - Docker') {
      steps {
        parallel(
        	"Dependency Scan": {
        		sh "mvn dependency-check:check"
			},
		  // 	  "Trivy Scan":{
			//   	  sh "bash trivy-k8s.sh"
			// },
			"OPA Conftest":{
				sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
			} 	
      	)
			}
      // post {
      //   always {
      //     dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
      //   }
      // }
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

// added as a part of opa conftest for kubernetes manifests
    stage('Vulnerability Scan - Kubernetes') {
      steps {
        parallel(
          "OPA Scan": {
            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
          },
          "Kubesec Scan": {
            sh "bash kubesec-scan.sh"
          }
          // "Trivy Scan": {
          //   sh "bash trivy-k8s-scan.sh"
          // }
        )
      }
    }
// added as a part of first k8s deployment
      stage('Kubernetes Deployment - Dev') {
        steps {
          parallel(
            "Deployment": {
              withKubeConfig([credentialsId: 'kubeconfig']){
                // sh "sed -i 's#replace#shahabuddin7/numericapp:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
                // sh "kubectl apply -f k8s_deployment_service.yaml"
                sh "bash k8s-deployment.sh"
        }
        },
        "Rollout Status": {
            withKubeConfig([credentialsId: 'kubeconfig']){
              sh "bash k8s-deployment-rollout-status.sh"
        }
    }
  )
  }
}

    stage('Integration Tests - DEV') {
      steps {
        script {
          try {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash integration-test.sh"
            }
          } catch (e) {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "kubectl -n default rollout undo deploy ${deploymentName}"
            }
            throw e
          }
        }
      }
    }


    stage('OWASP ZAP - DAST') {
      steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
          sh 'bash zap.sh'
        }
    }
  }


  }
  post {
    always {
          junit 'target/surefire-reports/*.xml'
          jacoco execPattern: 'target/jacoco.exec'
          pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
          dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
    }
  }
}
