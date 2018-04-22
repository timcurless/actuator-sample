pipeline {
  agent {
    kubernetes {
      label 'buildPod'
      containerTemplate {
        name 'maven'
        image 'maven:3-alpine'
        ttyEnabled true
        command 'cat'
      }
      containerTemplate {
        name 'docker'
        image 'cloudbees/java-with-docker-client'
        ttyEnabled true
        command 'cat'
      }
    }
  }

  stages {
    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    stage('Build App') {
      steps {
        container('maven') {
          sh "mvn -v"
          sh "mvn package -B"
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }

    stage ('Build Container') {
      steps {
        container('docker') {
          withCredentials(usernamePassword(credentialsId: 'timcurless+cje_robo_acct', usernameVariable: 'QUAY_USER',
                            passwordVariable: 'QUAY_PASS'))
          sh "docker build -t quay.io/timcurless/actuator-sample"
          sh "docker login --username $QUAY_USER --password $QUAY_PASS quay.io"
          sh "docker push quay.io/timcurless/actuator-sample"
        }
      }
    }
  }

}
