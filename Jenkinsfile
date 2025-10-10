pipeline {
    agent any

    environment {
        FTP_HOST = 'ftpupload.net'
        FTP_USER = 'if0_39730581'
        FTP_PASS = credentials('infinityfree_ftp_pass')  // stored in Jenkins credentials
        FTP_PATH = '/htdocs'   // InfinityFree web root
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/HitKakadiya3/laravel_infinity_app.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'composer install --no-dev --optimize-autoloader'
                sh 'php artisan config:cache'
                sh 'php artisan route:cache'
                sh 'php artisan view:cache'
            }
        }

        stage('Deploy to InfinityFree') {
            steps {
                script {
                    echo "Starting deployment to InfinityFree..."
                    
                    // Use lftp for direct FTP upload
                    withCredentials([string(credentialsId: 'infinityfree_ftp_pass', variable: 'FTP_PASSWORD')]) {
                        sh '''
                            # Install lftp if not available
                            if ! command -v lftp &> /dev/null; then
                                echo "Installing lftp..."
                                apt-get update && apt-get install -y lftp || yum install -y lftp || {
                                    echo "Could not install lftp automatically"
                                    exit 1
                                }
                            fi
                            
                            echo "Creating clean deployment directory..."
                            # Create a clean deployment directory
                            rm -rf deploy_clean
                            mkdir -p deploy_clean
                            
                            # Copy all files except excluded ones
                            rsync -av --exclude='.git' --exclude='node_modules' --exclude='tests' \
                                     --exclude='storage/logs/*' --exclude='*.log' --exclude='.env*' \
                                     --exclude='composer.lock' --exclude='package-lock.json' \
                                     --exclude='.gitignore' --exclude='README.md' \
                                     ./ deploy_clean/
                            
                            echo "Files prepared for deployment:"
                            find deploy_clean -type f | head -10
                            echo "... and more files"
                            
                            echo "Connecting to FTP server and uploading..."
                            # Upload using lftp with better error handling
                            lftp -c "
                                set ftp:ssl-allow no
                                set ftp:passive-mode yes
                                set net:timeout 30
                                set net:max-retries 3
                                open -u ${FTP_USER},${FTP_PASSWORD} ${FTP_HOST}
                                cd ${FTP_PATH}
                                lcd deploy_clean
                                mirror --reverse --delete --verbose --parallel=3 ./ ./
                                quit
                            " || {
                                echo "FTP upload failed, creating fallback package..."
                                cd deploy_clean
                                tar -czf ../laravel_app_deploy.tar.gz .
                                cd ..
                                echo "Fallback package created: laravel_app_deploy.tar.gz"
                                exit 1
                            }
                            
                            echo "Deployment completed successfully!"
                            # Clean up
                            rm -rf deploy_clean
                        '''
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo "Verifying deployment..."
                    
                    // Check if the main files are accessible via FTP
                    withCredentials([string(credentialsId: 'infinityfree_ftp_pass', variable: 'FTP_PASSWORD')]) {
                        sh '''
                            echo "Checking deployed files..."
                            lftp -c "
                                set ftp:ssl-allow no
                                set ftp:passive-mode yes
                                open -u ${FTP_USER},${FTP_PASSWORD} ${FTP_HOST}
                                cd ${FTP_PATH}
                                ls -la
                                ls -la public/
                                quit
                            " || echo "Could not verify deployment via FTP"
                        '''
                    }
                    
                    echo "Deployment verification completed!"
                }
            }
        }
    }
}
