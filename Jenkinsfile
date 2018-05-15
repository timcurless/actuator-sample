#!/usr/bin/env groovy
def label = "worker-${UUID.randomUUID().toString()}"

podTemplate(
  label: label,
  containers: [
    containerTemplate(name: 'maven', image: 'maven:alpine', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'awseks', image: 'timcurless/aws-eks-api:v3', alwaysPullImage: true, command: 'cat', ttyEnabled: true)
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

      stage('Check Code Quality') {
        withSonarQubeEnv('SonarQube Scanner') {
          sh 'mvn clean package sonar:sonar'
        }
      }

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
            credentialsId: 'EKS-API-User']])
          {
            sh """
              curl -O https://amazon-eks.s3-us-west-2.amazonaws.com/2018-04-04/eks-2017-11-01.normal.json
              aws configure add-model --service-model file://eks-2017-11-01.normal.json --service-name eks
            """
            sh 'export EKSNAME=$(aws eks describe-cluster --cluster-name eks-dev --region us-west-2 --query "cluster.clusterName") && sed -i -e \'s@<cluster-name>@\'"$EKSNAME"\'@g\' /root/.kube/config-eks'
            sh 'export EKSURL=$(aws eks describe-cluster --cluster-name eks-dev --region us-west-2 --query "cluster.masterEndpoint") && sed -i -e \'s@<endpoint-url>@\'"$EKSURL"\'@g\' /root/.kube/config-eks'
            sh 'export EKSKEY=$(aws eks describe-cluster --cluster-name eks-dev --region us-west-2 --query "cluster.certificateAuthority.data") && sed -i -e \'s@<base64-encoded-ca-cert>@\'"$EKSKEY"\'@g\' /root/.kube/config-eks'
            sh """
              export KUBECONFIG=/root/.kube/config-eks
              sed -i -e 's@<tag>@${gitCommit}@g' deployment.yml
              cat deployment.yml
              kubectl --namespace=default apply -f deployment.yml
            """
          }
        }
      }
    }
}
