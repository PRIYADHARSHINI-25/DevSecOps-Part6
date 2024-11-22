pipeline {
  agent any
  tools { 
        maven 'Maven_3_2_5'  
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=devops-sonar-priya_devsecops -Dsonar.organization=devops-sonar-priya -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=51804ce25c34b7240b58a5c8a4fd31e2744b3f8e'
			}
    }

	stage('RunSCAAnalysisUsingSnyk') {
            steps {		
				withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
					sh 'mvn snyk:test -fn'
				}
			}
    }	
	   
	stage('Build') { 
            steps { 
               withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                 script{
                 app =  docker.build("myimg")
                 }
               }
            }
    }

	stage('Push') {
            steps {
                script{
                    docker.withRegistry('https://741448956481.dkr.ecr.us-east-1.amazonaws.com', 'ecr:us-east-1:aws-credentials') {
                    app.push("latest")
                    }
                }
            }
    	}
	   
	stage('Kubernetes Deployment of EasyBugg Web Application') {
	   steps {
	      withKubeConfig([credentialsId: 'kubelogin']) {
		  sh('kubectl delete all --all -n devsecops-priya')
		  sh ('kubectl apply -f deployment.yaml --namespace=devsecops-priya')
		}
	      }
   	}
	   
	stage ('wait_for_testing'){
	   steps {
		   sh 'pwd; sleep 180; echo "Application has been deployed on K8S"'
	   	}
	   }

	stage('RunDASTUsingZAP') {
    	steps {
        withKubeConfig([credentialsId: 'kubelogin']) {
            sh('zap.sh -cmd -quickurl http://$(kubectl get services/easybuggy --namespace=devsecops-priya -o json | jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.json')

            script {
                def criticalAlerts = sh(
                    script: "cat ${WORKSPACE}/zap_report.json | jq -c '.site[].alerts[] | select(.risk == \"High\")'",
                    returnStdout: true
                ).trim()

                if (criticalAlerts) {
                    withCredentials([usernamePassword(credentialsId: 'jiralogin', usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_TOKEN')]) {
                        criticalAlerts.split('\n').each { alert ->
                            def summary = sh(script: "echo '${alert}' | jq -r '.alert'", returnStdout: true).trim()
                            def description = sh(script: "echo '${alert}' | jq -r '.description'", returnStdout: true).trim()
                            
                            // Call Jira REST API with secure credentials
                            sh """curl -X POST -u $JIRA_USER:$JIRA_TOKEN \
                                 -H "Content-Type: application/json" \
                                 -d '{
                                      "fields": {
                                          "project": {"key": "EAS"},
                                          "summary": "Critical Security Alert: ${summary}",
                                          "description": "${description}",
                                          "issuetype": {"name": "Bug"},
                                          "priority": {"name": "High"}
                                      }
                                  }' \
                                 https://priya35.atlassian.net/rest/api/2/issue"""
                        }
                    }
                }
            }
        }
    }
}

	   
// 	stage('RunDASTUsingZAP') {
//           steps {
// 		    withKubeConfig([credentialsId: 'kubelogin']) {
// 				sh('zap.sh -cmd -quickurl http://$(kubectl get services/easybuggy --namespace=devsecops-priya -o json| jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html')
// 				archiveArtifacts artifacts: 'zap_report.html'
// 		    }
// 	     }
//        } 
//   }
}
}
