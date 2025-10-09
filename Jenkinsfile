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
      agent { label 'build-cli' }
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
          echo "Running CPU unit tests..."
        '''
      }
    }

stage('GPU Stage (nested container)') {
  agent { label 'gpu' }
  steps {
    script {
      // Verify image
      sh 'docker inspect -f . elm/jenkins-agent:gpu || (echo "GPU image missing!" && exit 1)'

      // Run with proper mount and GPU access
      docker.image('elm/jenkins-agent:gpu').inside(
        '--gpus all --entrypoint= ' +
        "-v /var/run/docker.sock:/var/run/docker.sock " +
        "-v ${pwd()}:/home/jenkins/agent/workspace/Test3_master"
      ) {
        sh '''
          echo "Inside nested GPU container:"
          whoami
          java -version
          nvidia-smi | head -n 10 || echo "No nvidia-smi found"
          echo "Running CUDA/PyTorch test:"
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
      node('build-cli') {
        sh 'docker image prune -f || true'
      }
    }
  }
}
