pipeline {
  agent {
    kubernetes {
      label 'buildPod'
      defaultContainer: 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    pipeline: bagapi-pipeline
spec:
  containers:
  - name: maven
    image: maven:3-alpine
    command:
    - cat
    tty: true
  - name: docker
    image: cloudbees/java-with-docker-client
    command:
    - cat
    tty: true
"""
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
