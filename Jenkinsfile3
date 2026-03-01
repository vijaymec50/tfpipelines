pipeline {
  agent {
    kubernetes {
      label 'tf-agent'
      defaultContainer 'terraform'
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-agent
  containers:
  - name: terraform
    image: hashicorp/terraform:1.14.6
    command: ['cat']
    tty: true
"""
    }
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Terraform') {
      steps {
        dir('envs/dev') {
          sh '''
            terraform version
            terraform fmt -check -recursive
            terraform init -input=false
            terraform validate
            terraform plan -input=false
          '''
        }
      }
    }
  }
}
