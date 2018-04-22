#!/usr/bin/env groovy
def label = "worker-${UUID.randomUUID().toString()}"

podTemplate(
  label: label,
  containers [
    containerTemplate(name: 'maven', image: 'maven:alpine', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true)
  ],
  volumes: [
    hostPathToVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
  ]) {
    node(label) {
      def myRepo = checkout scm
      def gitCommit = myRepo.GIT_COMMIT

      stage('Build App') {
        container('maven') {
          sh "mvn package -B"
        }
      }

      stage('Build Docker Image') {
        container('docker') {
          withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: 'timcurless+cje_robo_acct',
            usernameVariable: 'QUAY_USER',
            passwordVariable: 'QUAY_PASS']]) {
              sh """
                docker login --username ${QUAY_USER} --password ${QUAY_PASS}
                docker build -t quay.io/timcurless/actuator-sample:${gitCommit} .
                docker push quay.io/timcurless/actuator-sample:${gitCommit}
              """
            }
        }
      }
    }
  }
)
