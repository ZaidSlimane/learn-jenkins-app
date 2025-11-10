pipeline {
  agent any

  options {
    // optional but nice to have
    timestamps()
  }

  stages {
    stage('Pre-pull Docker images') {
      steps {
        sh '''
          set -euxo pipefail
          # Pull once so later stages don't pay the cold-pull penalty
          docker pull node:18-alpine
          docker pull mcr.microsoft.com/playwright:v1.42.0-focal
          docker images | grep -E "node\\s+18-alpine|playwright\\s+v1.42.0-focal" || true
        '''
      }
    }

    stage('Build') {
      agent {
        docker {
          image 'node:18-alpine'
          reuseNode true
        }
      }
      steps {
        sh '''
          echo "=== BUILDING INSIDE DOCKER ==="
          npm ci
          npm run build
          ls -la build/
        '''
      }
    }

    stage('Test') {
      parallel {
        stage('Junit test') {
          agent {
            docker {
              image 'node:18-alpine'
              reuseNode true
            }
          }
          steps {
            sh 'npm test'
          }
          post {
            always {
              junit 'test-results/junit.xml'
            }
          }
        }

        stage('E2E') {
          agent {
            docker {
              image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
              reuseNode true
            }
          }
          steps {
            sh '''
              npx serve -s build &
              npx playwright test --reporter=html
            '''
          }
          post {
            always {
              publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: false,
                reportDir: 'playwright-report',
                reportFiles: 'index.html',
                reportName: 'Playwright E2E Report'
              ])
            }
          }
        }
      }
    }

    stage('Deploy') {
      agent {
        docker {
          image 'node:18-alpine'
          reuseNode true
        }
      }
      steps {
        sh '''
          npm install netlify-cli@20.1.1
          node_modules/.bin/netlify --version
        '''
      }
    }
  }
}
