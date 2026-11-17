pipeline {
  agent any
  options { timestamps() }

  environment {
    IMAGE = 'walabenrgaya/monapp'
    TAG   = "build-${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }  // lit le meme repo que le job
    }

    stage('Docker Build') {
      steps {
        bat 'whoami'
        bat 'docker version'
        bat "docker build -t %IMAGE%:%TAG% ."
      }
    }

    stage('Smoke Test') {
      steps {
        bat """
          docker rm -f monapp_test 2>nul || ver > nul
          docker run -d --name monapp_test -p 8082:80 %IMAGE%:%TAG%
          ping -n 3 127.0.0.1 > nul
          curl -I http://localhost:8082 | find "200 OK"
          docker rm -f monapp_test
        """
      }
    }

    stage('Push (Docker Hub)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'USER',
                                          passwordVariable: 'PASS')]) {
          bat """
            echo %PASS%| docker login -u %USER% --password-stdin || exit /b 1
            docker tag %IMAGE%:%TAG% %IMAGE%:latest
            docker push %IMAGE%:%TAG%
            docker push %IMAGE%:latest
          """
        }
      }
    }
    stage('Deploy (Staging)') {
      steps {
        bat """
          docker compose pull
          docker compose up -d
          docker ps --filter "name=monapp-web"
          ping -n 3 127.0.0.1 > nul
          curl -I http://localhost:8090 | find "200 OK"
        """
      }
    }
  }

  post {
    success { echo 'Build+Test+Push+Deploy OK' }
    failure { echo 'Build/Tests/Push/Deploy KO' }
  }
}
