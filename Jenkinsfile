pipeline {
    agent any

    // ─── Environment Variables ────────────────────────────────────────────
    environment {
        DEPLOY_DIR     = '/usr/share/nginx/html'
        BACKUP_DIR     = '/var/backups/website'
        WEBSITE_SOURCE = 'website'           // subfolder inside the repo
        NGINX_SERVICE  = 'nginx'
        BUILD_TIMESTAMP = sh(
            script: 'date +%Y%m%d_%H%M%S',
            returnStdout: true
        ).trim()
    }

    // ─── Pipeline Options ─────────────────────────────────────────────────
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    // ─── Stages ──────────────────────────────────────────────────────────
    stages {

        // 1. CHECKOUT
        stage('Checkout') {
            steps {
                echo "╔══════════════════════════════════╗"
                echo "║  Stage 1: Pulling Latest Code    ║"
                echo "╚══════════════════════════════════╝"
                checkout scm
                sh 'echo "Workspace: $(pwd)"'
                sh 'echo "Branch: $(git rev-parse --abbrev-ref HEAD)"'
                sh 'echo "Commit: $(git log -1 --pretty=format:"%h - %s (%an, %ar)")"'
            }
        }

        // 2. VALIDATE
        stage('Validate') {
            steps {
                echo "╔══════════════════════════════════╗"
                echo "║  Stage 2: Validating Files       ║"
                echo "╚══════════════════════════════════╝"
                script {
                    // Verify required files exist in the repo
                    def requiredFiles = [
                        "${WEBSITE_SOURCE}/index.html",
                        "${WEBSITE_SOURCE}/css/style.css",
                        "${WEBSITE_SOURCE}/js/app.js"
                    ]
                    requiredFiles.each { filePath ->
                        if (!fileExists(filePath)) {
                            error("❌ Required file missing: ${filePath}")
                        } else {
                            echo "✅ Found: ${filePath}"
                        }
                    }

                    // Validate index.html has minimum content
                    def htmlContent = readFile("${WEBSITE_SOURCE}/index.html")
                    if (!htmlContent.contains('<!DOCTYPE html>') &&
                        !htmlContent.contains('<!doctype html>')) {
                        error("❌ index.html does not appear to be a valid HTML file")
                    }
                    echo "✅ HTML validation passed"
                    echo "✅ All required files validated successfully"
                }
            }
        }

        // 3. BACKUP
        stage('Backup') {
            steps {
                echo "╔══════════════════════════════════╗"
                echo "║  Stage 3: Backing Up Website     ║"
                echo "╚══════════════════════════════════╝"
                script {
                    sh """
                        # Create backup directory if it doesn't exist
                        mkdir -p ${BACKUP_DIR}

                        # Only backup if deploy directory has content
                        if [ "\$(ls -A ${DEPLOY_DIR} 2>/dev/null)" ]; then
                            BACKUP_PATH="${BACKUP_DIR}/backup_${BUILD_TIMESTAMP}"
                            echo "📦 Creating backup at: \${BACKUP_PATH}"
                            cp -r ${DEPLOY_DIR} \${BACKUP_PATH}
                            echo "✅ Backup created: \${BACKUP_PATH}"

                            # Keep only the last 5 backups
                            echo "🧹 Cleaning old backups (keeping last 5)..."
                            ls -dt ${BACKUP_DIR}/backup_* 2>/dev/null | tail -n +6 | xargs rm -rf
                            echo "✅ Old backups cleaned"
                        else
                            echo "ℹ️  Deploy directory empty — skipping backup"
                        fi
                    """
                }
            }
        }

        // 4. DEPLOY
        stage('Deploy') {
            steps {
                echo "╔══════════════════════════════════╗"
                echo "║  Stage 4: Deploying Website      ║"
                echo "╚══════════════════════════════════╝"
                script {
                    sh """
                        echo "🚀 Starting deployment..."

                        # Remove old files from deploy directory
                        echo "🗑️  Clearing old website files..."
                        rm -rf ${DEPLOY_DIR}/*

                        # Copy new website files
                        echo "📂 Copying new files to ${DEPLOY_DIR}..."
                        cp -r ${WEBSITE_SOURCE}/* ${DEPLOY_DIR}/

                        # Set correct ownership and permissions
                        echo "🔒 Setting file permissions..."
                        chown -R jenkins:jenkins ${DEPLOY_DIR}
                        chmod -R 755 ${DEPLOY_DIR}
                        find ${DEPLOY_DIR} -type f -name "*.html" -exec chmod 644 {} \\;
                        find ${DEPLOY_DIR} -type f -name "*.css"  -exec chmod 644 {} \\;
                        find ${DEPLOY_DIR} -type f -name "*.js"   -exec chmod 644 {} \\;

                        echo "✅ Files deployed:"
                        ls -lh ${DEPLOY_DIR}
                    """
                }
            }
        }

        // 5. RESTART WEB SERVER
        stage('Restart Web Server') {
            steps {
                echo "╔══════════════════════════════════╗"
                echo "║  Stage 5: Restarting Nginx       ║"
                echo "╚══════════════════════════════════╝"
                script {
                    sh """
                        echo "🔍 Testing Nginx configuration..."
                        sudo nginx -t

                        echo "🔄 Reloading Nginx..."
                        sudo systemctl reload ${NGINX_SERVICE}

                        echo "✅ Nginx reloaded successfully"
                        sudo systemctl status ${NGINX_SERVICE} --no-pager
                    """
                }
            }
        }

        // 6. VERIFY
        stage('Verify Deployment') {
            steps {
                echo "╔══════════════════════════════════╗"
                echo "║  Stage 6: Verifying Deployment   ║"
                echo "╚══════════════════════════════════╝"
                script {
                    sh """
                        echo "⏳ Waiting 3 seconds for Nginx to settle..."
                        sleep 3

                        # Check index.html is present
                        if [ -f "${DEPLOY_DIR}/index.html" ]; then
                            echo "✅ index.html present in deploy directory"
                        else
                            echo "❌ index.html NOT found in ${DEPLOY_DIR}"
                            exit 1
                        fi

                        # HTTP health check via localhost
                        HTTP_STATUS=\$(curl -o /dev/null -s -w "%{http_code}" http://localhost/)
                        echo "🌐 HTTP Status Code: \${HTTP_STATUS}"

                        if [ "\${HTTP_STATUS}" = "200" ]; then
                            echo "✅ Website is live and returning HTTP 200"
                        else
                            echo "❌ Website check failed. HTTP Status: \${HTTP_STATUS}"
                            exit 1
                        fi

                        echo ""
                        echo "╔══════════════════════════════════════╗"
                        echo "║  ✅ DEPLOYMENT SUCCESSFUL!           ║"
                        echo "╚══════════════════════════════════════╝"
                    """
                }
            }
        }
    }

    // ─── Post Actions ─────────────────────────────────────────────────────
    post {
        success {
            echo "✅ Pipeline completed successfully! Build #${env.BUILD_NUMBER}"
            echo "🌐 Website: http://<YOUR_EC2_PUBLIC_IP>/"
            echo "🔧 Jenkins: http://<YOUR_EC2_PUBLIC_IP>:8080/"
        }
        failure {
            echo "❌ Pipeline FAILED at stage. Check logs above."
            echo "🔄 Attempting automatic rollback..."
            script {
                sh """
                    LATEST_BACKUP=\$(ls -dt ${BACKUP_DIR}/backup_* 2>/dev/null | head -1)
                    if [ -n "\${LATEST_BACKUP}" ]; then
                        echo "♻️  Rolling back to: \${LATEST_BACKUP}"
                        rm -rf ${DEPLOY_DIR}/*
                        cp -r \${LATEST_BACKUP}/* ${DEPLOY_DIR}/
                        sudo systemctl reload nginx
                        echo "✅ Rollback completed successfully"
                    else
                        echo "⚠️  No backup found — rollback skipped"
                    fi
                """
            }
        }
        always {
            echo "📋 Build #${env.BUILD_NUMBER} | Duration: ${currentBuild.durationString}"
            cleanWs()   // Clean workspace after every build
        }
    }
}
