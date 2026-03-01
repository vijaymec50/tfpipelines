// Jenkinsfile (production-grade pattern) — Kubernetes agents + GitHub + Terraform (S3 backend via init args)
//
// Assumptions / prerequisites in Jenkins:
// 1) Job is "Pipeline script from SCM" (so checkout scm works).
// 2) Kubernetes Cloud is configured and working (you already passed Test Connection).
// 3) Credentials:
//    - AWS credentials stored as Jenkins credential ID: aws-jenkins  (type: "AWS Credentials" OR use Secret Text/Username+Password and adjust)
//    - (Optional) GitHub creds if repo is private (configured in the job SCM section).
//
// Repo layout expected:
// repo-root/
//   infra/
//     vpc/
//       *.tf
//     rds/
//       *.tf
//   Jenkinsfile

pipeline {
  agent {
    kubernetes {
      // Unique label per build so multiple builds don’t collide
      label "tf-k8s-${env.BUILD_NUMBER}"
      // The default container for steps that don't specify container()
      defaultContainer 'tools'

      // Inline Pod spec so the pipeline is portable
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-tf-agent
spec:
  serviceAccountName: jenkins-agent
  securityContext:
    runAsUser: 1000
    fsGroup: 1000
  containers:
    - name: tools
      image: alpine/git:2.45.2
      command: ['cat']
      tty: true
    - name: terraform
      image: hashicorp/terraform:1.14.6
      command: ['cat']
      tty: true
      env:
        - name: TF_IN_AUTOMATION
          value: "true"
        - name: TF_INPUT
          value: "0"
"""
    }
  }

  options {
    timestamps()
    disableConcurrentBuilds()              // avoids state lock contention if same module/env
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 45, unit: 'MINUTES')
  }

  parameters {
    choice(name: 'INFRA_FOLDER', choices: ['infra'], description: 'Top-level infra folder (keep as infra unless you add more)')
    string(name: 'RESOURCE_NAME', defaultValue: 'vpc', description: 'Folder under infra/ (example: vpc, rds, eks)')
    choice(name: 'ENV', choices: ['dev', 'prod'], description: 'Environment name')
    choice(name: 'ACTION', choices: ['plan', 'apply'], description: 'plan = only plan. apply = plan + approval + apply')
    booleanParam(name: 'DESTROY', defaultValue: false, description: 'If true, plan/apply will be destroy (guarded by approval)')
  }

  environment {
    // Backend settings (centralized; no duplication in tf folders)
    TF_BACKEND_BUCKET = 'vj-my-tf-state-unique'      // <-- change me
    TF_BACKEND_REGION = 'us-east-1'               // <-- change me
    TF_BACKEND_LOCK_TABLE = 'terraform-locks'     // <-- change me

    // Derived paths
    TF_DIR = "${params.INFRA_FOLDER}/${params.RESOURCE_NAME}"
    TF_STATE_KEY = "${params.INFRA_FOLDER}/${params.RESOURCE_NAME}/${params.ENV}/terraform.tfstate"

    // Helpful for providers and modules
    AWS_DEFAULT_REGION = "${TF_BACKEND_REGION}"
  }

  stages {

    stage('Preflight') {
      steps {
        container('tools') {
          sh '''
            set -e
            echo "Build: ${JOB_NAME} #${BUILD_NUMBER}"
            echo "TF_DIR=${TF_DIR}"
            echo "TF_STATE_KEY=${TF_STATE_KEY}"
          '''
        }
      }
    }

    stage('Checkout') {
      steps {
        container('tools') {
          checkout scm
          sh '''
            set -e
            echo "Workspace contents (top):"
            ls -la
            echo "Terraform files under ${TF_DIR}:"
            ls -la "${TF_DIR}" || true
            find "${TF_DIR}" -maxdepth 2 -type f -name "*.tf" -print || true
          '''
        }
      }
    }

    stage('Fmt / Validate') {
      steps {
        container('terraform') {
          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-jenkins',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
          ]]) {
            dir("${TF_DIR}") {
              sh '''
                set -e
                terraform version
                terraform fmt -check -recursive

                # Backend config is passed here; backend.tf in folder should only contain: terraform { backend "s3" {} }
                terraform init -input=false \
                  -backend-config="bucket=${TF_BACKEND_BUCKET}" \
                  -backend-config="region=${TF_BACKEND_REGION}" \
                  -backend-config="dynamodb_table=${TF_BACKEND_LOCK_TABLE}" \
                  -backend-config="encrypt=true" \
                  -backend-config="key=${TF_STATE_KEY}"

                terraform validate
              '''
            }
          }
        }
      }
    }

    stage('Plan') {
      steps {
        container('terraform') {
          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-jenkins',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
          ]]) {
            dir("${TF_DIR}") {
              sh '''
                set -e

                # Re-init is safe and makes the step standalone
                terraform init -input=false \
                  -backend-config="bucket=${TF_BACKEND_BUCKET}" \
                  -backend-config="region=${TF_BACKEND_REGION}" \
                  -backend-config="dynamodb_table=${TF_BACKEND_LOCK_TABLE}" \
                  -backend-config="encrypt=true" \
                  -backend-config="key=${TF_STATE_KEY}"

                if [ "${DESTROY}" = "true" ]; then
                  echo "Running DESTROY plan..."
                  terraform plan -input=false -destroy -out=tfplan \
                    -var="environment=${ENV}"
                else
                  terraform plan -input=false -out=tfplan \
                    -var="environment=${ENV}"
                fi

                terraform show -no-color tfplan > tfplan.txt
              '''
            }
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: "${TF_DIR}/tfplan,${TF_DIR}/tfplan.txt", allowEmptyArchive: true
        }
      }
    }

    stage('Approval') {
      when {
        expression { return params.ACTION == 'apply' }
      }
      steps {
        script {
          // Safety gates:
          // - Require main branch for prod applies
          // - Require manual approval for any apply (including dev)
          def branch = env.BRANCH_NAME ?: "unknown"
          def isProd = (params.ENV == 'prod')
          if (isProd && branch != 'main') {
            error("Blocked: prod apply is only allowed from main branch. Current branch: ${branch}")
          }

          def actionLabel = params.DESTROY ? "DESTROY" : "APPLY"
          input message: "${actionLabel} ${params.ENV} for ${params.RESOURCE_NAME}?\nState key: ${env.TF_STATE_KEY}",
                ok: "Yes, ${actionLabel}"
        }
      }
    }

    stage('Apply') {
      when {
        expression { return params.ACTION == 'apply' }
      }
      steps {
        container('terraform') {
          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-jenkins',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
          ]]) {
            dir("${TF_DIR}") {
              sh '''
                set -e

                terraform init -input=false \
                  -backend-config="bucket=${TF_BACKEND_BUCKET}" \
                  -backend-config="region=${TF_BACKEND_REGION}" \
                  -backend-config="dynamodb_table=${TF_BACKEND_LOCK_TABLE}" \
                  -backend-config="encrypt=true" \
                  -backend-config="key=${TF_STATE_KEY}"

                # Apply the saved plan for immutability
                terraform apply -input=false -auto-approve tfplan
              '''
            }
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Success: ${params.ACTION} completed for ${params.RESOURCE_NAME} (${params.ENV})."
    }
    failure {
      echo "❌ Failed: check console logs; common causes: wrong TF_DIR, missing .tf files, backend perms, or DNS/Jenkins URL issues."
    }
  }
}
