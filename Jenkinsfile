pipeline {
  agent any

  options {
    disableConcurrentBuilds()
  }

  environment {
    COV_LINES      = '80'
    COV_BRANCHES   = '70'
    COV_FUNCTIONS  = '75'
    COV_STATEMENTS = '80'
    CI = 'true'
    JEST_JUNIT_OUTPUT = 'test-results/junit.xml'
  }

  stages {
    stage('Checkout') {
      steps {
        git url: 'https://github.com/<user>/<repo>.git', branch: 'main'
      }
    }

    stage('Build') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        sh '''
          set -e
          node -v && npm -v

          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install
          fi

          npm run build || true
          ls -la
        '''
      }
    }

    stage('Test') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        sh '''
          set -e
          if [ -f package-lock.json ]; then npm ci; else npm install; fi

          mkdir -p test-results
          npm test -- --ci --watchAll=false --coverage --testResultsProcessor=jest-junit --passWithNoTests
        '''
      }
      post {
        always {
          junit allowEmptyResults: false, testResults: 'test-results/junit.xml'
        }
      }
    }

    stage('Quality Gate (Coverage)') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        sh '''
          set -e
          if [ ! -f coverage/coverage-summary.json ]; then
            echo "coverage-summary.json not found. Did tests run with --coverage?"
            exit 1
          fi

          node -e "
            const fs = require('fs');
            const thr = {
              lines:      +process.env.COV_LINES,
              branches:   +process.env.COV_BRANCHES,
              functions:  +process.env.COV_FUNCTIONS,
              statements: +process.env.COV_STATEMENTS,
            };
            const total = JSON.parse(fs.readFileSync('coverage/coverage-summary.json','utf8')).total;
            const pct = {
              lines: total.lines.pct,
              branches: total.branches.pct,
              functions: total.functions.pct,
              statements: total.statements.pct,
            };
            console.log('Coverage:', pct, '\\nThresholds:', thr);
            const failed = Object.keys(thr).filter(k => (pct[k] ?? 0) < thr[k]);
            if (failed.length) {
              console.error('Quality gate FAILED for:', failed.map(k => `${k} ${pct[k]}% < ${thr[k]}%`).join(', '));
              process.exit(1);
            } else {
              console.log('Quality gate PASSED ✅');
            }
          "
        '''
      }
    }

    stage('Check Build Output') {
      agent { docker { image 'node:18-alpine'; reuseNode true } }
      steps {
        sh '''
          set -e
          echo ">> Verifying build/ exists and is not empty..."
          if [ -d build ] && [ "$(ls -A build)" ]; then
            echo "✅ build/ exists and is not empty"
            ls -la build
          else
            echo "❌ build/ missing or empty!"
            exit 1
          fi
        '''
      }
    }
  }
}
