node{
	stage ("Get tools version") {
		Maven_Version = sh (script: 'ls /var/jenkins_home/tools/hudson.tasks.Maven_MavenInstallation/', returnStdout: true).trim()
		JAVA_Version = sh (script: 'ls /var/jenkins_home/tools/hudson.model.JDK/', returnStdout: true).trim()
	}
}
pipeline {
agent any
	parameters {
		choice(name: 'MavenVersion', choices: "${Maven_Version}", description: 'On this step you need select Maven Version')
		choice(name: 'JavaVersion', choices: "${JAVA_Version}", description: 'On this step you need select JAVA Version')
		choice(name: 'Deploing', choices: "YES\nNO", description: 'Option for allow/decline deploy to ENV')
		choice(name: 'ENVIRONMENT', choices: "TEST\nSTAGE\nPROD", description: 'Please select ENV server for deploy you APP')
		choice(name: 'TomcatVersion', choices: "Tomcat7\nTomcat8\nTomcat9", description: 'Please select ENV server for deploy you APP')

	}
	environment {
		sh (script: 'ls ./vars', returnStdout: true).trim()
	}
	tools {
		maven "${params.MavenVersion}"
		jdk "${params.JavaVersion}"
	}
stages {
	stage('Version Java&Tomcat compatibility check') {
		when { 
			anyOf {
				expression { params.JavaVersion == 'Java7' && params.TomcatVersion == 'Tomcat9' };
				expression { params.JavaVersion == 'Java8' && params.TomcatVersion == 'Tomcat7' };
				expression { params.JavaVersion == 'Java9' && params.TomcatVersion == 'Tomcat7' }
			}	
		}
		steps {
			sh 'echo "ERROR: ${TomcatVersion} does not support ${JavaVersion}" && exit 1'
		}
	}
	stage('Get Source Code && Build With maven') {
		steps {
			script {
				sh "git clone $GIT_SOURCE"
				sh "cp -r ./$JOB_NAME/* ./"
				sh "mv Dockerfile_${JavaVersion}_${TomcatVersion} Dockerfile"
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
					url: "${artifactory_url}",
					username: "${artifactory_user}",
					password: "${artifactory_password}"
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
				sh "sed -i s/BUILD_NUMBER/${BUILD_NUMBER}/g docker-compose.yaml"
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
	stage('Deploing image to ENV') {
		when { 
				expression { params.Deploing == 'YES' }
		}
		steps {
			sh "scp -o StrictHostKeyChecking=no ./docker-compose.yaml root@${env."${params.ENVIRONMENT}"}:/root/"
			sh "ssh -o StrictHostKeyChecking=no root@${env."${params.ENVIRONMENT}"} 'docker-compose up --build -d'"
		}
	}
	stage('Remove Unused docker image') {
		steps{
			sh "docker rmi -f $registry/$JOB_NAME:latest"
			sh "docker rmi -f $registry/$JOB_NAME:v$BUILD_NUMBER"
		}
	}
}
	post {
		always {
			echo 'One way or another, I have finished'
			deleteDir() /* clean up our workspace */
		}
	}
}
