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
		STAGE = '192.168.23.7'
		TEST = '192.168.23.7'
		PROD = '192.168.23.7'
		registry = 'registry.domain.com:5000'
		dockerImage = ''
		APP_EXTPORT = '30100'
		GIT_SOURCE = "https://github.com/werdervg/${JOB_NAME}.git"
		replace_registry_path='$registry/$JOB_NAME:v$BUILD_NUMBER'
		Maven_OPTS = '-Dmaven.test.failure.ignore'
		artifactory_user = 'publisheruser'
		artifactory_password = 'pa@sswo2rd1'
		artifactory_url = 'http://artifactory:8081/artifactory'
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
				sh 'echo ${${params.ENVIRONMENT}}'
				sh "exit 1"
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
	stage('Deploing image to STAGE ENV') {
		when { 
				expression { params.Deploing == 'YES' && params.ENVIRONMENT == 'STAGE' }
		}
		steps {
			sh "scp -o StrictHostKeyChecking=no ./docker-compose.yaml root@$STAGE:/root/"
			sh "ssh -o StrictHostKeyChecking=no root@$STAGE 'docker-compose up --build -d'"
		}
	}
	stage('Deploing image to TEST ENV') {
		when { 
			expression { params.Deploing == 'YES' && params.ENVIRONMENT == 'TEST'}
		}
		steps {
			sh "scp -o StrictHostKeyChecking=no ./docker-compose.yaml root@$TEST:/root/"
			sh "ssh -o StrictHostKeyChecking=no root@$TEST 'docker-compose up --build -d'"
		}
	}
	stage('Deploing image to PROD ENV') {
		when { 
			expression { params.Deploing == 'YES' && params.ENVIRONMENT == 'PROD' }
		}
		steps {
			sh "scp -o StrictHostKeyChecking=no ./docker-compose.yaml root@$PROD:/root/"
			sh "ssh -o StrictHostKeyChecking=no root@$PROD 'docker-compose up --build -d'"
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
