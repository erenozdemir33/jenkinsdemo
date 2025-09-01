pipeline {
  /* Tek container / tek workspace: tüm stage'ler aynı Node 18 Alpine içinde koşar */
  agent {
    docker {
      image 'node:18-alpine'
      args '-u 0:0'           // root user: paket kurulumları için kolaylık
      reuseNode true
    }
  }

  options {
    disableConcurrentBuilds()
    timestamps()
  }

  environment {
    COV_LINES      = '80'
    COV_BRANCHES   = '70'
    COV_FUNCTIONS  = '75'
    COV_STATEMENTS = '80'
    CI = 'true'
    // jest-junit bu env'i otomatik kullanır
    JEST_JUNIT_OUTPUT = 'test-results/junit.xml'
  }

  stages {
    stage('Checkout') {
      steps {
        // Kaynak: Jenkins job'ı SCM bağlıysa 'checkout scm' yeterli
        checkout scm
      }
    }

    stage('Deps & Build') {
      steps {
        sh '''
          set -e
          node -v && npm -v

          # Kilit dosyasına göre install
          if [ -f package-lock.json ] || [ -f npm-shrinkwrap.json ]; then
            npm ci
          else
            npm install
          fi

          # Build
          npm run build || true
          ls -la
        '''
      }
    }

    stage('Test') {
      steps {
        sh '''
          set -e

          # jest-junit yoksa devDependency olarak ekle (pipeline ortamında garanti)
          if ! npm ls --depth=0 jest-junit >/dev/null 2>&1; then
            npm i -D jest-junit
          fi

          mkdir -p test-results

          # CRA -> react-scripts test Jest argümanlarını geçirir
          # Reporter yaklaşımı (güncel/pratik): junit raporunu üretir
          npm test -- \
            --ci \
            --watchAll=false \
            --coverage \
            --reporters=default \
            --reporters=jest-junit \
            --passWithNoTests

          # Debug amaçlı göster
          if [ -f test-results/junit.xml ]; then
            echo "JUnit report generated ✅ -> test-results/junit.xml"
          else
            echo "JUnit report missing (allowed to be empty)."
          fi
        '''
      }
      post {
        // Rapor üretilememişse pipeline'ı hemen kırmasın; sonraki coverage gate karar verir
        always {
          junit allowEmptyResults: true, testResults: 'test-results/junit.xml'
        }
      }
    }

    stage('Quality Gate (Coverage)') {
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
