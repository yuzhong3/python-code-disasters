pipeline {
  agent any

  triggers {
    githubPush()
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

                export CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE="$GOOGLE_APPLICATION_CREDENTIALS"
                gcloud auth activate-service-account --key-file "$GOOGLE_APPLICATION_CREDENTIALS"
                gcloud config set project "$PROJECT_ID"
                gcloud config set dataproc/region "$REGION"

                TO_COPY=(README.md sonar-project.properties Jenkinsfile django flask python obfuscation)
                gcloud storage rm -r "gs://$BUCKET/input/repo" || true
                for p in "${TO_COPY[@]}"; do
                if [ -e "$p" ]; then
                    gcloud storage cp -r "$p" "gs://$BUCKET/input/repo/"
                fi
                done

                cat > mapper.py << 'PY'
        import os, sys
        fname = (os.environ.get("mapreduce_map_input_file")
                or os.environ.get("map_input_file")
                or "unknown")
        prefix = "/input/repo/"
        i = fname.find(prefix)
        if i != -1:
            name = fname[i + len(prefix):]
        else:
            name = os.path.basename(fname)
        for _ in sys.stdin:
            print(f"\\"{name}\\"\\t1")
        PY

                cat > reducer.py << 'PY'
        import sys
        current = None
        total = 0
        def flush(k, v):
            if k is not None:
                print(f"{k}\\t{v}")
        for line in sys.stdin:
            line = line.rstrip("\\n")
            if not line:
                continue
            key, val = line.split("\\t", 1)
            if key != current:
                flush(current, total)
                current = key
                total = 0
            total += int(val)
        flush(current, total)
        PY

                gcloud storage cp mapper.py "gs://$BUCKET/jobs/mapper.py"
                gcloud storage cp reducer.py "gs://$BUCKET/jobs/reducer.py"

                OUT="${OUT_DIR}"
                gcloud dataproc jobs submit hadoop \
                --cluster="$CLUSTER" \
                --jar=file:///usr/lib/hadoop-mapreduce/hadoop-streaming.jar \
                -- \
                -D mapreduce.input.fileinputformat.input.dir.recursive=true \
                -files "gs://$BUCKET/jobs/mapper.py,gs://$BUCKET/jobs/reducer.py" \
                -mapper "python3 mapper.py" \
                -reducer "python3 reducer.py" \
                -numReduceTasks 1 \
                -input "gs://$BUCKET/input/repo/**" \
                -output "$OUT"

                echo "[INFO] Submitted. Output at: $OUT"
                gcloud storage ls "$OUT" || true
                echo "[INFO] Preview (first 20 lines):"
                gcloud storage cat "$OUT/part-00000" | head -20 || true
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