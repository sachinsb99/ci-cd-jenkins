pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sachinsb99/ci-cd-jenkins.git'
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                mkdir -p /var/www/html/static-site
                cp -r *.html *.css /var/www/html/static-site/
                '''
            }
        }
    }

    post {
        success {
            echo 'Deployment completed successfully.'
        }
    }
}
