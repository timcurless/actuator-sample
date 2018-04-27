#!/usr/bin/env groovy
def label = "worker-${UUID.randomUUID().toString()}"

podTemplate(
  label: label,
  containers: [
    containerTemplate(name: 'maven', image: 'maven:alpine', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'awseks', image: 'timcurless/aws-eks-api:v2', alwaysPullImage: true, command: 'cat', ttyEnabled: true)
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
  ]) {
    node(label) {
      properties(
        [
            buildDiscarder(
                logRotator(
                    numToKeepStr: '7'
                )
            )
        ]
      )
      def myRepo = checkout scm
      def gitCommit = myRepo.GIT_COMMIT

      stage('Build App') {
        container('maven') {
          sh "mvn package -B"
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }

      stage('Build Docker Image') {
        container('docker') {
          withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: '0515b9d5-2cc5-4ef4-a355-9cc55ab47ffd',
            usernameVariable: 'QUAY_USER',
            passwordVariable: 'QUAY_PASS']])
          {
            sh """
              docker login --username ${QUAY_USER} --password ${QUAY_PASS} quay.io
              docker build -t quay.io/timcurless/actuator-sample:${gitCommit} .
              docker push quay.io/timcurless/actuator-sample:${gitCommit}
            """
          }
        }
      }

      stage('Deploy App') {
        container('awseks') {
          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'EKS-API-User',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']])
          {
            sh """
              cat <<EOT > /root/.aws/credentials\\
              [default]\\
              aws_access_key_id=${AWS_ACCESS_KEY_ID}\\
              aws_secret_access_key=${AWS_SECRET_ACCESS_KEY}\\
              EOT\\
              cat /root/.aws/credentials
              aws eks describe-cluster --cluster-name eks-dev --region us-west-2
            """
          }
        }
      }
    }
}
