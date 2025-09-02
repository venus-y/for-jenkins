pipeline {
  agent any

  environment {
    imagename          = 'venus-y/for-jenkins'   // <dockerhub-id>/<repo>
    registryCredential = 'DOCKER-HUB'            // Jenkins Docker Hub 자격증명 ID
    dockerImage        = ''
  }

  stages {
    stage('Prepare') {
      steps {
        echo 'Cloning Repository'
        git url: 'git@github.com:venus-y/for-jenkins.git',
            branch: 'main',
            credentialsId: 'rsa'
      }
      post {
        success { echo 'Successfully Cloned Repository' }
        failure { error 'This pipeline stops here...' }
      }
    }

    stage('Build Gradle') {
      steps {
        echo 'Build Gradle'
        sh 'chmod +x ./gradlew || true'
        sh './gradlew clean build -x test'
      }
      post {
        failure { error 'This pipeline stops here...' }
      }
    }

    stage('Build Docker') {
      steps {
        echo 'Build Docker'
        script {
          def tag = sh(script: "git rev-parse --short=12 HEAD || echo ${env.BUILD_NUMBER}", returnStdout: true).trim()
          env.IMAGE_TAG = tag
          dockerImage = docker.build("${imagename}:${env.IMAGE_TAG}")
        }
      }
      post {
        failure { error 'This pipeline stops here...' }
      }
    }

    stage('Push Docker') {
      steps {
        echo 'Push Docker'
        script {
          docker.withRegistry('', registryCredential) {
            dockerImage.push()                                         // <imagename>:<IMAGE_TAG>
            sh "docker tag ${imagename}:${env.IMAGE_TAG} ${imagename}:latest"
            sh "docker push ${imagename}:latest"
          }
        }
      }
      post {
        failure { error 'This pipeline stops here...' }
      }
    }

    stage('Docker Run') {
      steps {
        echo 'Pull Docker Image & Run on Remote'
        sshagent (credentials: ['spring-ssh']) {
          sh """
            ssh -o StrictHostKeyChecking=no ubuntu@203.0.113.10 \\
              'docker pull ${imagename}:${env.IMAGE_TAG} &&
               (docker ps -q --filter name=myapp | grep -q . && docker rm -f myapp || true) &&
               (docker ps -aq --filter name=myapp | grep -q . && docker rm -f myapp || true) &&
               docker run -d --name myapp -p 8080:8080 --restart=always ${imagename}:${env.IMAGE_TAG}'
          """
        }
      }
    }
  }
}
