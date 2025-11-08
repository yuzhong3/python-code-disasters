pipeline {
  agent any
  triggers { githubPush() }
  options { disableConcurrentBuilds(); timestamps() }

  environment {
    SONARQUBE = 'sonar'
    PROJECT_ID = 'project-1-474119'
    REGION    = 'us-central1'
    CLUSTER   = 'cluster-6185'
    BUCKET    = 'week6-eurus-test1'
  }

  stages {
    stage('Clean workspace') {
      steps { deleteDir() }
    }

    stage('Checkout') { 
        steps { checkout scm } 
    }

    stage('SonarQube Analysis') {
      steps {
        script {
          def scannerHome = tool 'sonar-scanner'
          withSonarQubeEnv("${SONARQUBE}") {
            sh "${scannerHome}/bin/sonar-scanner"
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    // —— 用 Kubernetes 动态起一个带 gcloud 的 Pod 跑该阶段 —— //
    stage('Submit Dataproc Job') {
      agent {
        kubernetes {
          defaultContainer 'gcloud'
          yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: default
  restartPolicy: Never
  containers:
  - name: gcloud
    image: google/cloud-sdk:slim
    command: ['cat']
    tty: true
"""
        }
      }
      steps {
        withCredentials([file(credentialsId: 'gcp-sa', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
          sh '''
            set -euo pipefail
            gcloud --version
            gcloud auth activate-service-account --key-file "$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud config set project "$PROJECT_ID"
            gcloud config set dataproc/region "$REGION"

            echo "hello cloud hadoop dataproc test" > sample.txt
            gsutil cp sample.txt gs://$BUCKET/input/sample.txt || true

            OUT=gs://$BUCKET/output/w6-${BUILD_NUMBER}-$(date +%s)
            gcloud dataproc jobs submit hadoop \
              --cluster="$CLUSTER" \
              --class=org.apache.hadoop.examples.WordCount \
              -- \
              -Dmapreduce.input.fileinputformat.input.dir.recursive=true \
              gs://$BUCKET/input/** \
              "$OUT"

            echo "Output at: $OUT"
            gsutil ls "$OUT" || true
          '''
        }
      }
    }
  }
}