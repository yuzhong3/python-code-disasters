pipeline {
  agent any

  triggers {
    githubPush()
  }

  options {
    disableConcurrentBuilds()
  }

  environment {
    SONARQUBE = 'sonar'
    PROJECT_ID = 'course-project-option-1'
    REGION = 'us-central1'
    CLUSTER = 'hadoop-cluster'
    BUCKET = 'course-project-option-1-hadoop-data-4d06d748'
    OUT_DIR = "gs://${BUCKET}/output/run-${env.BUILD_NUMBER}-${new Date().getTime()}"
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
            set -exu

            SDK_VER=google-cloud-cli-502.0.0-linux-x86_64
            TARBALL="${SDK_VER}.tar.gz"
            URL="https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/${TARBALL}"
            curl -sSLO "$URL"
            tar -xf "$TARBALL"

            GCLOUD_BIN="$(find "$PWD" -maxdepth 4 -type f -path '*/google-cloud-sdk/bin/gcloud' | head -1)"
            if [ -z "$GCLOUD_BIN" ] || [ ! -x "$GCLOUD_BIN" ]; then
            echo "[ERROR] gcloud not found after extraction"
            echo "[DEBUG] PWD=$PWD"
            ls -la
            exit 127
            fi
            export PATH="$(dirname "$GCLOUD_BIN"):$PATH"
            "$GCLOUD_BIN" --version
            which gcloud; gcloud --version

            export CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE="$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud auth activate-service-account --key-file "$GOOGLE_APPLICATION_CREDENTIALS"
            gcloud config set project "$PROJECT_ID"
            gcloud config set dataproc/region "$REGION"

            TO_COPY="README.md sonar-project.properties Jenkinsfile django flask python obfuscation"
            gcloud storage rm -r "gs://$BUCKET/input/repo" || true
            for p in $TO_COPY; do
            if [ -e "$p" ]; then
                gcloud storage cp -r "$p" "gs://$BUCKET/input/repo/"
            fi
            done

            curl -sSLo hadoop-streaming.jar \
            https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-streaming/3.3.6/hadoop-streaming-3.3.6.jar
            gcloud storage cp hadoop-streaming.jar "gs://$BUCKET/jobs/hadoop-streaming-3.3.6.jar"

            printf '%s\n' \
            'import os, sys' \
            'fname = (os.environ.get("mapreduce_map_input_file") or os.environ.get("map_input_file") or "unknown")' \
            'prefix = "/input/repo/"' \
            'i = fname.find(prefix)' \
            'if i != -1:' \
            '    name = fname[i + len(prefix):]' \
            'else:' \
            '    name = os.path.basename(fname)' \
            'for _ in sys.stdin:' \
            '    print(f"\\"{name}\\"\\t1")' \
            > mapper.py

            printf '%s\n' \
            'import sys' \
            'current = None' \
            'total = 0' \
            'def flush(k, v):' \
            '    if k is not None:' \
            '        print(f"{k}\\t{v}")' \
            'for line in sys.stdin:' \
            '    line = line.rstrip("\\n")' \
            '    if not line:' \
            '        continue' \
            '    key, val = line.split("\\t", 1)' \
            '    if key != current:' \
            '        flush(current, total)' \
            '        current = key' \
            '        total = 0' \
            '    total += int(val)' \
            'flush(current, total)' \
            > reducer.py

            gcloud storage cp mapper.py "gs://$BUCKET/jobs/mapper.py"
            gcloud storage cp reducer.py "gs://$BUCKET/jobs/reducer.py"

            OUT="${OUT_DIR}"

            echo "[INFO] Submitting Dataproc Streaming job..."
            JOB_ID=$(gcloud dataproc jobs submit hadoop \
            --cluster="$CLUSTER" \
            --region="$REGION" \
            --async \
            --format='get(reference.jobId)' \
            --jar="gs://$BUCKET/jobs/hadoop-streaming-3.3.6.jar" \
            -- \
            -D mapreduce.input.fileinputformat.input.dir.recursive=true \
            -files "gs://$BUCKET/jobs/mapper.py,gs://$BUCKET/jobs/reducer.py" \
            -mapper "python3 mapper.py" \
            -reducer "python3 reducer.py" \
            -numReduceTasks 1 \
            -input "gs://$BUCKET/input/repo/**" \
            -output "$OUT")

            echo "[INFO] Submitted Dataproc job: $JOB_ID. Waiting..."
            gcloud dataproc jobs wait "$JOB_ID" --region="$REGION"

            echo "[INFO] Output should be at: $OUT"
            gcloud storage ls "$OUT"
            echo "[INFO] Preview (first 20 lines):"
            gcloud storage cat "$OUT/part-00000" | head -20

            gcloud storage cat "$OUT/part-00000" | awk -F '\\t' '{print $1": "$2}' > summary.txt
            gcloud storage cp summary.txt "gs://$BUCKET/output/$(basename "$OUT")/summary.txt" || true
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