pipeline {
  agent any

  // 触发：GitHub push（可另加 pollSCM 兜底）
  triggers {
    githubPush()
    // pollSCM('H/5 * * * *')  // 可选兜底：每5分钟轮询一次
  }

  options {
    disableConcurrentBuilds()       // 防止并发
    timestamps()
  }

  environment {
    SONARQUBE = 'sonar'
    PROJECT_ID = 'project-1-474119'
    REGION     = 'us-central1'
    CLUSTER    = 'cluster-6185'
    BUCKET     = 'week6-eurus-test1'  // TODO: 改成你的实际 bucket
    // 输出路径自动带时间戳
    OUT_DIR    = "gs://${BUCKET}/output/w6-${env.BUILD_NUMBER}-${new Date().getTime()}"
  }

  stages {
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
            set -e
            export PATH="/usr/lib/google-cloud-sdk/bin:$PATH"

            gcloud auth activate-service-account --key-file "$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud config set project ${PROJECT_ID}
            gcloud config set dataproc/region ${REGION}

            echo "hello cloud hadoop dataproc test test" > sample.txt
            gsutil cp sample.txt gs://${BUCKET}/input/sample.txt || true

            gcloud dataproc jobs submit hadoop \
              --cluster=${CLUSTER} \
              --class=org.apache.hadoop.examples.WordCount \
              -- \
              -Dmapreduce.input.fileinputformat.input.dir.recursive=true \
              gs://${BUCKET}/input/** \
              ${OUT_DIR}

            echo "Submitted Dataproc job. Output: ${OUT_DIR}"
            gsutil ls ${OUT_DIR} || true
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