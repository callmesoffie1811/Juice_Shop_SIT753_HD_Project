pipeline {
  agent any
  options { timestamps() }
  tools { nodejs 'nodejs_lts' }

  environment {
    APP_NAME   = 'juice-shop'
    STAGING_TAG = "${APP_NAME}:staging"
    PROD_TAG    = "${APP_NAME}:prod"
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build') {
      steps {
        sh '''
          echo "Installing deps"
          npm ci || npm install
          echo "Building Docker image (staging)"
          docker build -t ${STAGING_TAG} . || true
        '''
      }
    }

    stage('Test') {
      steps {
        sh '''
          echo "Running tests"
          mkdir -p reports/junit
          npx mocha --reporter mocha-junit-reporter --reporter-options mochaFile=reports/junit/results.xml || true
          npm test || true
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'reports/junit/*.xml'
        }
      }
    }

    stage('Code Quality') {
      steps {
        sh '''
          echo "ESLint checkstyle report"
          mkdir -p reports/eslint
          npx eslint . -f checkstyle -o reports/eslint/eslint.xml || true
        '''
      }
      post {
        always {
          // Requires "Warnings" (Warnings NG) plugin
          recordIssues enabledForFailure: true, tools: [checkStyle(pattern: 'reports/eslint/*.xml')]
        }
      }
    }

    stage('Security') {
      steps {
        sh '''
          echo "npm audit (dependency scan)"
          npm audit --audit-level=high || true

          echo "Trivy image scan"
          if ! command -v trivy >/dev/null; then
            curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin || true
          fi
          docker image inspect ${STAGING_TAG} >/dev/null 2>&1 && \
            trivy image --exit-code 0 --severity LOW,MEDIUM ${STAGING_TAG} || true
          docker image inspect ${STAGING_TAG} >/dev/null 2>&1 && \
            trivy image --exit-code 1 --severity HIGH,CRITICAL ${STAGING_TAG} || true
        '''
      }
    }

    stage('Deploy: Staging') {
      steps {
        sh '''
          cat > docker-compose.yml <<'YML'
          services:
            juice-staging:
              image: ${STAGING_TAG}
              build: .
              ports: ["3000:3000"]
              healthcheck:
                test: ["CMD-SHELL","curl -fsS http://localhost:3000 || exit 1"]
                interval: 10s
                retries: 12
            juice-prod:
              image: ${PROD_TAG}
              ports: ["3001:3000"]
              healthcheck:
                test: ["CMD-SHELL","curl -fsS http://localhost:3000 || exit 1"]
                interval: 10s
                retries: 12
            cadvisor:
              image: gcr.io/cadvisor/cadvisor:latest
              ports: ["8081:8080"]
              volumes:
                - /:/rootfs:ro
                - /var/run/docker.sock:/var/run/docker.sock:ro
                - /var/lib/docker/:/var/lib/docker:ro
          YML

          docker compose down || true
          docker compose build juice-staging || true
          docker compose up -d juice-staging cadvisor || true

          echo "Waiting for staging health..."
          for i in $(seq 1 30); do
            curl -fsS http://localhost:3000 >/dev/null && echo "Staging OK" && exit 0
            sleep 2
          done
          echo "Staging not healthy yet (continuing for demo)"
          exit 0
        '''
      }
    }

    stage('Release: Promote to Prod') {
      steps {
        sh '''
          docker tag ${STAGING_TAG} ${PROD_TAG} || true
          docker compose up -d juice-prod || true

          echo "Waiting for prod health..."
          for i in $(seq 1 30); do
            curl -fsS http://localhost:3001 >/dev/null && echo "Prod OK" && exit 0
            sleep 2
          done
          echo "Prod not healthy yet (continuing for demo)"
          exit 0
        '''
      }
    }

    stage('Monitoring & Alerting') {
      steps {
        sh '''
          echo "Simple latency check against prod"
          START=$(date +%s%3N); curl -fsS -o /dev/null http://localhost:3001 || true
          RT=$(( $(date +%s%3N) - START ))
          echo "Prod response time (ms): $RT"
          echo "Open cAdvisor at http://localhost:8081 for container metrics"
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
    }
  }
}
