pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }
    stage('Check Build Output') {
        steps {
            sh '''
                echo ">> Checking if build directory was created..."

                # Test if the "build" folder exists
                if [ -d build ]; then
                    echo "Build directory exists."
                    echo "Listing build directory content:"
                    ls -la build
                else
                    echo "Build directory does not exist!"
                    # Uncomment the next line if you want the pipeline to fail
                    exit 1
                fi
            '''
        }
    }
    }
}
