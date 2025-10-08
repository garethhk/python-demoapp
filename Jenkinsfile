pipeline {
  agent { label 'build-cli' }   // << run most stages in ONE container/workspace
  options {
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout(true)   // we call 'checkout scm' explicitly
  }
  environment {
    DOCKER_BUILDKIT = '1'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD > .git/shortsha || true'
      }
    }

    stage('Compute Version') {
      steps {
        script {
          def shortSha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          def ts = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
          env.BUILD_VERSION = "${shortSha}-${ts}"
        }
        echo "Build version: ${env.BUILD_VERSION}"
      }
    }

    stage('Detect stack') {
      steps {
        sh '''
          echo "Node:   $(node --version 2>/dev/null || echo 'missing')"
          echo "Python: $(python3 --version 2>/dev/null || echo 'missing')"
          echo "Java:   $(java -version 2>&1 | head -n1 || echo 'missing')"
          echo "Docker: $(docker --version 2>/dev/null || echo 'missing')"
        '''
      }
    }

    stage('Python: install & test') {
      when { expression { fileExists('requirements.txt') } }
      steps {
        sh '''
          python3 -m venv .venv
          . .venv/bin/activate
          pip install -U pip
          pip install -r requirements.txt
          if [ -f pytest.ini ] || ls -1 tests/*.py >/dev/null 2>&1; then
            python -m pytest -q
          else
            echo "No pytest config/tests found. Skipping."
          fi
        '''
      }
      post { always { sh 'rm -rf .venv || true' } }
    }

    stage('Node: install & test') {
      when { expression { fileExists('package.json') } }
      steps {
        sh '''
          npm ci || npm install
          if jq -re '.scripts.test' package.json >/dev/null 2>&1; then
            npm test
          else
            echo "No npm test script. Skipping."
          fi
        '''
      }
    }

    stage('GPU sanity (optional)') {
      agent { label 'gpu' }   // only this stage uses the GPU agent
      when { expression { false } } // flip to true when you want to test GPU
      steps {
        sh '''
          nvidia-smi || true
          python3 - <<'PY'
try:
  import torch
  print("Torch:", torch.__version__)
  print("CUDA available:", torch.cuda.is_available())
except Exception as e:
  print("torch not present or failed:", e)
PY
        '''
      }
    }

    stage('Build container image') {
      when { expression { fileExists('Dockerfile') } }
      steps {
        sh '''
          docker build -t myapp:${BUILD_VERSION} .
          docker image ls | head -n 20
        '''
      }
    }

    stage('Push image (optional)') {
      when { expression { false } } // set true and configure your registry creds/URL
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'DOCKER_USER',
                                          passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker tag myapp:${BUILD_VERSION} myorg/myapp:${BUILD_VERSION}
            docker push myorg/myapp:${BUILD_VERSION}
          '''
        }
      }
    }

    stage('Cleanup (local cache)') {
      steps {
        sh 'docker image prune -f || true'
      }
    }
  }

  post {
    always {
      echo 'Pipeline finished.'
    }
  }
}
