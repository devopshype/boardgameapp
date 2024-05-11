pipeline {
agent any
tools {
	jdk 'Java17'
	maven 'Maven3'
 }
environment {
	SCANNER_HOME=tool 'sonar-scanner'
	dockerRegistry="047075577663.dkr.ecr.ap-south-1.amazonaws.com"
	ecrRepo="boardgame"
	dockerImageTag="version-${BUILD_NUMBER}"
	helmchartname="boardgame"
	}

stages {

	stage('Gitcheckout'){
	agent {label 'docker'}
		steps {
			checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'githubCred', url: 'https://github.com/devopshype/boardgameapp.git']])
			}
		}
	
	stage('SonarAnalysis'){
	agent {label 'docker'}
	steps {
		withSonarQubeEnv('sonar-server') {
			sh '''
            mvn clean verify sonar:sonar \
            -Dsonar.projectName="$ecrRepo" \
            -Dsonar.projectKey="$ecrRepo" \
	    -Dsonar.java.binaries=. \
     	    -Dsonar.profile=myQualityProfile \
            -Dsonar.report.export.path=sonar-report.json'''
			}
		}
	}

	stage('QualityGate'){
	agent {label 'docker'}
	steps{
		script {
			waitForQualityGate abortPipeline: false, credentialsId: 'sonarToken'
			}
		}
	}
	
	stage('Push to Nexus'){
	agent {label 'docker'}
	steps{
		withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'Java17', maven: 'Maven3', mavenSettingsConfig: '', traceability: true) {
			sh "mvn clean deploy"
			}
		}
	}
	
	stage('Imagebuild'){
	agent {label 'docker'}
	steps{
		sh "docker build -t ${dockerRegistry}/${ecrRepo}:${dockerImageTag} ."
		sh "docker tag ${dockerRegistry}/${ecrRepo}:${dockerImageTag} ${dockerRegistry}/${ecrRepo}:latest"
		}
	}
	
	stage('Docker Image Scan') {
	agent {label 'docker'}
	steps {
		sh "trivy image --format table -o trivy-image-report.html ${dockerRegistry}/${ecrRepo}:${dockerImageTag} "
		}
	}

	stage('login to AWS ECR & ImagePush'){
	agent {label 'docker'}
	steps{
		script{
			withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'awsCredID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
				sh "aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin ${dockerRegistry}"
				sh "docker push ${dockerRegistry}/${ecrRepo}:${dockerImageTag}"
				sh "docker push ${dockerRegistry}/${ecrRepo}:latest"
			}
		  }
	    }
	 }
	
	stage('Kubernetes Gitcheckout'){
	agent {label 'k8s'}
		steps {
			checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'githubCred', url: 'https://github.com/devopshype/boardgameapp.git']])
			}
		}
		
	stage('Deploy to Kubernetes'){
	agent {label 'k8s'}
	steps{
		sh "sudo helm upgrade --install ${helmChartName} helm/boardgame/ --set image.repository=${dockerRegistry}/${ecrRepo}:${dockerImageTag}"
			}
		}
	
   }
}
