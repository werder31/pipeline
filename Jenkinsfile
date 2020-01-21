node{
	stage ("Get tools version") {
		Maven_Version = sh (script: 'ls /var/jenkins_home/tools/hudson.tasks.Maven_MavenInstallation/', returnStdout: true).trim()
		JAVA_Version = sh (script: 'ls /var/jenkins_home/tools/hudson.model.JDK/', returnStdout: true).trim()
	}
	stage ("Get JAVA version") {
		JAVA_Version = sh (script: 'ls /var/jenkins_home/tools/hudson.model.JDK/', returnStdout: true).trim()
	}
}
pipeline {
agent any
	parameters {
		choice(name: 'MavenVersion', choices: "${Maven_Version}", description: 'On this step you need select Maven Version')
		choice(name: 'JavaVersion', choices: "${JAVA_Version}", description: 'On this step you need select JAVA Version')
		choice(name: 'Deploing', choices: "NO\nYES", description: 'Option for allow/decline deploy to ENV')
		choice(name: 'ENVIRONMENT', choices: "STAGE\nTEST\nPROD", description: 'Please select ENV server for deploy you APP')
	}
	environment {
		STAGE = '192.168.23.7'
		TEST = '192.168.23.7'
		PROD = '192.168.23.7'
		registry = 'registry.domain.com:5000'
		dockerImage = ''
		GIT_SOURCE = 'https://github.com/werdervg/$JOB_NAME.git'
		APP_EXTPORT = '30100'
		replace_registry_path='$registry/$JOB_NAME:v$BUILD_NUMBER'
		Maven_OPTS = '-Dmaven.test.failure.ignore'
	}
	tools {
		maven "${params.MavenVersion}"
		jdk "${params.JavaVersion}"
	}
stages {
	stage('Build With maven') {
		steps {
			script {
				sh "mv Dockerfile_$JavaVersion Dockerfile"
				sh "mvn $Maven_OPTS clean package"
				sh 'cp target/*.jar app.jar'
			}
		}
	}
	stage('push_to_Artifactory') {
		steps {
			script{
				rtServer (
					id: 'Artifactory-1',
					url: 'http://artifactory:8081/artifactory',
					username: 'publisheruser',
					password: 'pa@sswo2rd1'
//					credentialsId: 'firstrepo'
					)
				rtUpload (
					serverId: 'Artifactory-1',
					spec: '''{
					"files": [
						{
						"pattern": "target/*.jar",
						"target": "$JOB_NAME/"
						}
					]
					}''',
					buildName: "$JOB_NAME",
					buildNumber: "$BUILD_NUMBER")
			}
		}	
	}
	stage('Building image and preparing compose file') {
		steps{
			script {
				sh "sed -i s/#build/build/g docker-compose.yaml"
				sh "sed -i s/JOB_NAME/${JOB_NAME}/g docker-compose.yaml"
				sh "sed -i s/APP_NAME/${JOB_NAME}/g docker-compose.yaml"
				sh "sed -i s/APP_EXTPORT/${APP_EXTPORT}/g docker-compose.yaml"
				sh "sed -i s/REGISTRY_NAME/${registry}/g docker-compose.yaml"
				dockerImage = docker.build registry + "/$JOB_NAME" + ":v$BUILD_NUMBER"
				sh "docker login https://$registry"
				sh "docker push $registry/$JOB_NAME:v$BUILD_NUMBER"
				sh "docker tag $registry/$JOB_NAME:v$BUILD_NUMBER $registry/$JOB_NAME:latest"
				sh "docker push $registry/$JOB_NAME:latest"
				sh "sed -i s/build/#build/g docker-compose.yaml"
				sh "sed -i s/#image/image/g docker-compose.yaml"
			}
		}
	}
	stage('Deploing image to STAGE ENV') {
		when { 
			allOf { 
				expression { params.Deploing == 'YES' }; 
				expression { params.ENVIRONMENT == 'STAGE' }
			}
		}
		steps {
			sh "echo DEPLOIIIIIIIIIIIIIIIG  TO STAGE"
			sh "scp -o StrictHostKeyChecking=no ./docker-compose.yaml root@$STAGE:/root/"
			sh "ssh -o StrictHostKeyChecking=no root@$STAGE 'docker-compose up --build -d'"
		}
	}
	stage('Deploing image to TEST ENV') {
		when { 
			allOf { 
				expression { params.Deploing == 'YES' }; 
				expression { params.ENVIRONMENT == 'TEST' }
			}
		}
		steps {
			sh "echo DEPLOIIIIIIIIIIIIIIIG  TO TEST"
			sh "scp -o StrictHostKeyChecking=no ./docker-compose.yaml root@$TEST:/root/"
			sh "ssh -o StrictHostKeyChecking=no root@$TEST 'docker-compose up --build -d'"
		}
	}
	stage('Deploing image to PROD ENV') {
		when { 
			allOf { 
				expression { params.Deploing == 'YES' }; 
				expression { params.ENVIRONMENT == 'PROD' }
			}
		}
		steps {
			sh "echo DEPLOIIIIIIIIIIIIIIIG  TO PROD"
			sh "scp -o StrictHostKeyChecking=no ./docker-compose.yaml root@$PROD:/root/"
			sh "ssh -o StrictHostKeyChecking=no root@$PROD 'docker-compose up --build -d'"
		}
	}
	stage('Remove Unused docker image') {
		steps{
			sh "docker rmi -f $registry/$JOB_NAME:latest"
			sh "docker rmi -f $registry/$JOB_NAME:v$BUILD_NUMBER"
			sh "rm -rf ./*"
		}
	}
}
}
