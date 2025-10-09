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
    // set by outer agent; inner GPU container gets its own env too
    DOCKER_BUILDKIT = '1'
  }

  stages {

    stage('Checkout') {
      agent { label 'build-cli' } // your working CPU agent
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD > .git/shortsha || true'
      }
    }

    stage('CPU Build & Test') {
      agent { label 'build-cli' }
      steps {
        sh '''
          echo "Node: $(node --version 2>/dev/null || echo missing)"
          echo "Python: $(python3 --version 2>/dev/null || echo missing)"
          echo "Docker: $(docker --version 2>/dev/null || echo missing)"
          echo "Java: $(java -version 2>&1 | head -n1 || echo missing)"

          # your normal build/test steps here
          echo "Running CPU unit tests..."
        '''
      }
    }

    stage('GPU Stage (nested container)') {
      agent { label 'build-cli' }
      when {
        anyOf {
          expression { return params.USE_GPU }                   // manual toggle
          branch pattern: '.*gpu.*', comparator: 'REGEXP'        // branch naming convention
          expression { return fileExists('gpu.requirements.txt') } // repo signal file
        }
      }
      steps {
        script {
          // Run inside your GPU image with --gpus all
          docker.image('elm/jenkins-agent:gpu').inside('--gpus all -v /var/run/docker.sock:/var/run/docker.sock') {
            withEnv([
              'NVIDIA_VISIBLE_DEVICES=all',
              'NVIDIA_DRIVER_CAPABILITIES=compute,utility'
            ]) {
              sh '''
                echo "Inside nested GPU container:"
                whoami
                java -version
                (nvidia-smi || echo "No GPU detected") | head -n 10

                # Example: Python GPU job (torch optional)
                python3 - <<'PY'
try:
    import torch
    print("Torch:", torch.__version__)
    print("CUDA available:", torch.cuda.is_available())
    if torch.cuda.is_available():
        print("GPU name:", torch.cuda.get_device_name(0))
except Exception as e:
    print("Torch not installed or failed:", e)
PY
              '''
            }
          }
        }
      }
    }

    stage('Build Container Image (optional)') {
      agent { label 'build-cli' }
      when { expression { fileExists('Dockerfile') } }
      steps {
        sh '''
          VERSION=$(cat .git/shortsha 2>/dev/null || echo dev)-$(date +%Y%m%d%H%M%S)
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
      // Optional lightweight cleanup on the outer agent
      node('build-cli') {
        sh 'docker image prune -f || true'
      }
    }
  }
}
