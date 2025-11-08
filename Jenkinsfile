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
    CLUSTER    = 'cluster-6185'
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

            echo "[INFO] Downloading ${TARBALL} ..."
            curl -sSLO "$URL"

            echo "[INFO] Extracting ..."
            tar -xf "$TARBALL"

            # 自动探测 gcloud 路径（两种常见布局）
            ROOT_DIR=""
            if [ -d "${SDK_VER}/google-cloud-sdk/bin" ]; then
            ROOT_DIR="${SDK_VER}/google-cloud-sdk"
            elif [ -d "google-cloud-sdk/bin" ]; then
            ROOT_DIR="google-cloud-sdk"
            else
            # 再兜底扫一遍
            ROOT_DIR=$(tar -tzf "$TARBALL" | head -1 | cut -d/ -f1)
            if [ -d "${ROOT_DIR}/google-cloud-sdk/bin" ]; then
                ROOT_DIR="${ROOT_DIR}/google-cloud-sdk"
            fi
            fi

            if [ ! -d "${ROOT_DIR}/bin" ]; then
            echo "[ERROR] Cannot locate google-cloud-sdk/bin after extraction."
            echo "[DEBUG] PWD=$(pwd)"; ls -la
            exit 127
            fi

            export PATH="$PWD/${ROOT_DIR}/bin:$PATH"
            echo "[INFO] Using gcloud from: $PWD/${ROOT_DIR}/bin"
            which gcloud
            gcloud --version

            # 认证 & 提交作业（按需替换你的变量）
            gcloud auth activate-service-account --key-file "$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud config set project project-1-474119
            gcloud config set dataproc/region us-central1

            BUCKET=week6-eurus-test1
            OUT_DIR="gs://${BUCKET}/output/w6-$(date +%s)"

            # 确保有输入
            echo "hello cloud hadoop dataproc test test" > sample.txt
            gsutil cp sample.txt gs://${BUCKET}/input/sample.txt || true

            gcloud dataproc jobs submit hadoop \
            --cluster=cluster-6185 \
            --class=org.apache.hadoop.examples.WordCount \
            -- \
            -Dmapreduce.input.fileinputformat.input.dir.recursive=true \
            gs://${BUCKET}/input/** \
            ${OUT_DIR}

            echo "[INFO] Submitted. Output at: ${OUT_DIR}"
            gsutil ls ${OUT_DIR} || true
        '''
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