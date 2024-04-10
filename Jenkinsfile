pipeline{
agent any
tools {
	jdk 'Java17'
	maven 'Maven3'
 }
environment {
	dockerRegistry="047075577663.dkr.ecr.ap-south-1.amazonaws.com"
	ecrRepo="boardgame"
	dockerImageTag="version-${BUILD_NUMBER}"
	helmChartName="boardgame"
	path="helm/boardgame"
 }
stages{
	stage('Gitcheckout'){
		parallel{
			stage('master'){
				steps{
					checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'gitcreds', url: 'https://github.com/devopshype/boardgameapp.git']])
					}
				}
			
			stage('kubernetes'){
				agent {label 'K8S'}
				steps{
				   checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'gitcreds', url: 'https://github.com/devopshype/boardgameapp.git']])
					}
				}
			}	
		}

  stage('Maven Build') {
	  steps {
		  sh "mvn package"
		}
	}
  
	stage('SonarAnalysis'){
	steps{
		withSonarQubeEnv('sonar') {
			sh '''mvn clean verify sonar:sonar \
				    -Dsonar.projectKey=BoardGame '''
			}
		}
	}

	stage('QualityGate'){
	steps{
		script {
			waitForQualityGate abortPipeline: false, credentialsId: 'sonartoken'
			}
		}
	}

	stage('Push to Nexus'){
	steps{
		withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'Java17', maven: 'Maven3', mavenSettingsConfig: '', traceability: true) {
			sh "mvn clean deploy"
			}
		}
	}
	
	stage('Imagebuild'){
	steps{
		sh "docker build -t ${dockerRegistry}/${ecrRepo}:${dockerImageTag} ."
		sh "docker tag ${dockerRegistry}/${ecrRepo}:${dockerImageTag ${dockerRegistry}/${ecrRepo}:latest"
		}
	}
	
	stage('Docker Image Scan') {
	steps {
		sh "trivy image --format table -o trivy-image-report.html ${dockerRegistry}/${ecrRepo}:${dockerImageTag} "
		}
	}

	stage('login to AWS ECR & ImagePush'){
	steps{
		script{
			withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'awscreds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
				sh "aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${dockerRegistry}"
				sh "docker push ${dockerRegistry}/${ecrRepo}:${dockerImageTag}"
				sh "docker push ${dockerRegistry}/${ecrRepo}:latest"
			}
		}
	    }
	}
	
	stage('Deploy to Kubernetes'){
	agent {label 'K8S'}
	steps{
		sh "helm upgrade --install ${helmChartName} ${path} --set image.repository=${dockerRegistry}/${ecrRepo} --set image.tag=${dockerImageTag}"
			}
		}
	}
}
