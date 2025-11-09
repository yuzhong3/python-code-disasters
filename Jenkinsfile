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
                set -exu

                # ---- 安装 Cloud SDK（自动探测路径） ----
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
                if [ -d "${ROOT_DIR}/google-cloud-sdk/bin" ]; then
                    ROOT_DIR="${ROOT_DIR}/google-cloud-sdk"
                fi
                fi
                export PATH="$PWD/${ROOT_DIR}/bin:$PATH"
                which gcloud; gcloud --version

                # ---- 强制 Cloud SDK 使用 Jenkins 的 SA JSON ----
                export CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE="$GOOGLE_APPLICATION_CREDENTIALS"
                gcloud auth activate-service-account --key-file "$GOOGLE_APPLICATION_CREDENTIALS"
                gcloud config set project "$PROJECT_ID"
                gcloud config set dataproc/region "$REGION"

                # ---- 仅拷贝源码白名单到 GCS（避免上传 google-cloud-sdk / 压缩包）----
                TO_COPY="README.md sonar-project.properties Jenkinsfile django flask python obfuscation"
                gcloud storage rm -r "gs://$BUCKET/input/repo" || true
                for p in $TO_COPY; do
                if [ -e "$p" ]; then
                    gcloud storage cp -r "$p" "gs://$BUCKET/input/repo/"
                fi
                done

                # ---- 生成 mapper/reducer（按仓库相对路径统计行数）----
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

                # ---- 提交 Streaming 作业 -> 获取 JobID -> 等待完成 -> 校验输出 ----
                OUT="${OUT_DIR}"

                echo "[INFO] Submitting Dataproc Streaming job..."
                # 使用公共 GCS JAR，适配所有 Dataproc 版本
                JOB_ID=$(gcloud dataproc jobs submit hadoop \
                --cluster="$CLUSTER" \
                --region="$REGION" \
                --async \
                --format='get(reference.jobId)' \
                --jar=gs://hadoop-lib/hadoop-mapreduce/hadoop-streaming.jar \
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

                # （可选）生成题目所需的美化版本
                awk -F '\\t' '{print $1": "$2}' <(gcloud storage cat "$OUT/part-00000") \
                | gcloud storage cp - "gs://$BUCKET/output/$(basename "$OUT")/summary.txt" || true
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