pipeline {
  agent none

  options {
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout(true)
  }

  parameters {
    booleanParam(name: 'USE_GPU', defaultValue: false, description: 'Run GPU stages in nested container')
  }

  environment {
    DOCKER_BUILDKIT = '1'
  }

  stages {
    stage('Checkout') {
      agent { label 'build-cli' }    // runs on your CPU build agent
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD > .git/shortsha || true'
      }
    }

    stage('CPU Build & Test') {
      agent { label 'build-cli' }
      steps {
        sh '''
          echo "Node:   $(node --version 2>/dev/null || echo missing)"
          echo "Python: $(python3 --version 2>/dev/null || echo missing)"
          echo "Docker: $(docker --version 2>/dev/null || echo missing)"
          echo "Java:   $(java -version 2>&1 | head -n1 || echo missing)"
          echo "Running CPU unit tests..."
          # put your real unit test commands here
        '''
      }
    }

    stage('GPU Stage (nested container)') {
      when { expression { return params.USE_GPU } }
      agent { label 'gpu' }          // runs on your GPU agent
      steps {
        script {
          // Verify the GPU agent image exists locally (fail fast if not)
          sh 'docker inspect -f . elm/jenkins-agent:gpu >/dev/null || (echo "GPU image elm/jenkins-agent:gpu missing!" && exit 1)'

          // IMPORTANT: do NOT mount the workspace yourself — Jenkins does it for you.
          // Only add GPU flag + docker.sock and clear entrypoint.
          docker.image('elm/jenkins-agent:gpu').inside('--gpus all --entrypoint= -v /var/run/docker.sock:/var/run/docker.sock') {
            sh '''
              echo "Inside nested GPU container:"
              whoami
              java -version

              echo "Checking GPU..."
              if command -v nvidia-smi >/dev/null 2>&1; then
                nvidia-smi | head -n 10
              else
                echo "No nvidia-smi found (no GPU runtime?)"
              fi

              echo "Python/CUDA quick check (optional)..."
              if command -v python3 >/dev/null 2>&1; then
                python3 - <<'PY'
try:
    import torch
    print("Torch:", torch.__version__)
    print("CUDA available:", torch.cuda.is_available())
    if torch.cuda.is_available():
        print("Device:", torch.cuda.get_device_name(0))
except Exception as e:
    print("Torch not installed or failed:", e)
PY
              else
                echo "python3 missing in GPU image; skipping torch check."
              fi
            '''
          }
        }
      }
    }

    stage('Build Container Image (optional)') {
      agent { label 'build-cli' }
      when { expression { fileExists('Dockerfile') } }
      steps {
        sh '''
          VERSION="$(cat .git/shortsha 2>/dev/null || echo dev)-$(date +%Y%m%d%H%M%S)"
          echo "Building image myapp:${VERSION}"
          docker build -t myapp:${VERSION} .
          docker image ls | head -n 20
        '''
      }
    }
  }

  post {
    always {
      echo 'Pipeline finished.'
      // Use a Docker-capable node for cleanup
      node('build-cli') {
        sh 'docker image prune -f || true'
      }
    }
  }
}
