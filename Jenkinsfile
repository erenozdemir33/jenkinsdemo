pipeline {
  agent {
    docker {
      image 'node:18-alpine'
      args '-u 0:0'         // root: kolay kurulum
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
    JEST_JUNIT_OUTPUT = 'test-results/junit.xml'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Deps & Build') {
      steps {
        sh '''
          set -e
          node -v && npm -v

          if [ -f package-lock.json ] || [ -f npm-shrinkwrap.json ]; then
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
      steps {
        sh '''
          set -e
          if ! npm ls --depth=0 jest-junit >/dev/null 2>&1; then
            npm i -D jest-junit
          fi

          mkdir -p test-results

          npm test -- \
            --ci \
            --watchAll=false \
            --coverage \
            --coverageReporters=json-summary \
            --reporters=default \
            --reporters=jest-junit \
            --passWithNoTests

          test -f test-results/junit.xml && echo "JUnit report generated ✅ -> test-results/junit.xml" || true
          test -f coverage/coverage-summary.json && echo "Coverage summary OK ✅" || true
        '''
      }
      post {
        always {
          // İstersen burada allowEmptyResults:false yapıp test raporu yoksa kırdırabilirsin
          junit allowEmptyResults: true, testResults: 'test-results/junit.xml'
        }
      }
    }

    stage('Quality Gate (Coverage)') {
      steps {
        sh '''
          set -e

          COV_SUMMARY="coverage/coverage-summary.json"
          COV_FINAL="coverage/coverage-final.json"

          # coverage-summary yoksa coverage-final'dan üret
          if [ ! -f "$COV_SUMMARY" ] && [ -f "$COV_FINAL" ]; then
            echo "coverage-summary.json yok; coverage-final.json kullanılacak (fallback)."
            node <<'NODE'
const fs = require('fs');
const total = JSON.parse(fs.readFileSync('coverage/coverage-final.json','utf8')).total;
const out = { total: {
  lines:      { pct: total.lines.pct },
  branches:   { pct: total.branches.pct },
  functions:  { pct: total.functions.pct },
  statements: { pct: total.statements.pct }
}};
fs.mkdirSync('coverage', { recursive: true });
fs.writeFileSync('coverage/coverage-summary.json', JSON.stringify(out, null, 2));
NODE
          fi

          if [ ! -f "$COV_SUMMARY" ]; then
            echo "coverage-summary.json not found. Did tests run with --coverage and json-summary?"
            exit 1
          fi

          # Eşik kontrolü — quoted HEREDOC ile güvenli
          node <<'NODE'
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
  console.error('Quality gate FAILED for: ' +
    failed.map(k => `${k} ${pct[k]}% < ${thr[k]}%`).join(', '));
  process.exit(1); // <-- Fail => pipeline FAIL, sonraki stage'ler çalışmaz
} else {
  console.log('Quality gate PASSED ✅');
}
NODE
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
