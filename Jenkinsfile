pipeline {
  agent any

  // 触发：GitHub push（可另加 pollSCM 兜底）
  triggers {
    githubPush()
    // pollSCM('H/5 * * * *')  // 可选兜底：每5分钟轮询一次
  }

  options {
    disableConcurrentBuilds()
    timestamps()
  }

  environment {
    SONARQUBE = 'sonar'
    PROJECT_ID = 'project-1-474119'
    REGION     = 'us-central1'
    CLUSTER    = 'cluster-w6'
    BUCKET     = 'week6-eurus-test1'
    OUT_DIR    = "gs://${BUCKET}/output/w6-${env.BUILD_NUMBER}-${new Date().getTime()}"
  }

  stages {
    stage('Clean workspace') {
      steps { deleteDir() }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
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

    stage('Submit Dataproc Job') {
        steps {
            withCredentials([file(credentialsId: 'gcp-sa', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
            sh '''
                set -euo pipefail

                SDK_VER=google-cloud-cli-502.0.0-linux-x86_64
                TARBALL="${SDK_VER}.tar.gz"
                URL="https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/${TARBALL}"
                curl -sSLO "$URL"
                tar -xf "$TARBALL"

                ROOT_DIR=""
                if [ -d "${SDK_VER}/google-cloud-sdk/bin" ]; then
                ROOT_DIR="${SDK_VER}/google-cloud-sdk"
                elif [ -d "google-cloud-sdk/bin" ]; then
                ROOT_DIR="google-cloud-sdk"
                else
                ROOT_DIR=$(tar -tzf "$TARBALL" | head -1 | cut -d/ -f1)
                [ -d "${ROOT_DIR}/google-cloud-sdk/bin" ] && ROOT_DIR="${ROOT_DIR}/google-cloud-sdk"
                fi
                export PATH="$PWD/${ROOT_DIR}/bin:$PATH"
                which gcloud; gcloud --version

                # 强制所有 Cloud SDK 命令使用这份 JSON 凭证（避免 metadata 凭证/受限 scope）
                export CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE="$GOOGLE_APPLICATION_CREDENTIALS"

                gcloud auth activate-service-account --key-file "$GOOGLE_APPLICATION_CREDENTIALS"
                gcloud config set project "$PROJECT_ID"
                gcloud config set dataproc/region "$REGION"

                # 确保有输入（用 gcloud storage，避免 gsutil 的 Boto 凭证坑）
                echo "hello cloud hadoop dataproc test test" > sample.txt
                gcloud storage cp sample.txt "gs://$BUCKET/input/sample.txt" || true

                # 如 cluster-6185 不健康，先换成新的 CLUSTER 名再跑
                OUT="${OUT_DIR}"
                gcloud dataproc jobs submit hadoop \
                --cluster="$CLUSTER" \
                --class=org.apache.hadoop.examples.WordCount \
                -- \
                -Dmapreduce.input.fileinputformat.input.dir.recursive=true \
                "gs://$BUCKET/input/**" \
                "$OUT"

                echo "[INFO] Submitted. Output at: $OUT"
                gcloud storage ls "$OUT" || true
            '''
            }
        }
    }
  }

  post {
    success {
      echo "Pipeline finished SUCCESS. Output at ${env.OUT_DIR}"
    }
    failure {
      echo "Pipeline FAILED - check console log and Sonar dashboard."
    }
  }
}