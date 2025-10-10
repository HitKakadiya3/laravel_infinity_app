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
                    
                    // Use curl for FTP upload (more universally available)
                    withCredentials([string(credentialsId: 'infinityfree_ftp_pass', variable: 'FTP_PASSWORD')]) {
                        sh '''
                            echo "Creating clean deployment directory..."
                            # Create a clean deployment directory
                            rm -rf deploy_clean
                            mkdir -p deploy_clean
                            
                            # Copy all files except excluded ones
                            echo "Copying files for deployment..."
                            find . -type f -not -path "./.git/*" -not -path "./node_modules/*" -not -path "./tests/*" \
                                   -not -path "./storage/logs/*" -not -name "*.log" -not -name ".env*" \
                                   -not -name "composer.lock" -not -name "package-lock.json" \
                                   -not -name ".gitignore" -not -name "README.md" -not -name "Jenkinsfile" \
                                   -exec cp --parents {} deploy_clean/ \\;
                            
                            echo "Files prepared for deployment:"
                            find deploy_clean -type f | head -10
                            echo "... and $(find deploy_clean -type f | wc -l) total files"
                            
                            echo "Uploading files using curl..."
                            
                            # Function to upload a single file
                            upload_file() {
                                local file="$1"
                                local remote_path="$2"
                                local remote_dir=$(dirname "$remote_path")
                                
                                # Create directory if it doesn't exist
                                curl -s --ftp-create-dirs -T /dev/null "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}${remote_dir}/" || true
                                
                                # Upload the file
                                if curl -T "$file" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}${remote_path}"; then
                                    echo "✓ Uploaded: $remote_path"
                                else
                                    echo "✗ Failed: $remote_path"
                                    return 1
                                fi
                            }
                            
                            # Upload all files
                            cd deploy_clean
                            failed_uploads=0
                            total_files=$(find . -type f | wc -l)
                            current_file=0
                            
                            find . -type f | while read file; do
                                current_file=$((current_file + 1))
                                remote_path="${file#./}"
                                echo "[$current_file/$total_files] Uploading: $remote_path"
                                
                                if ! upload_file "$file" "/$remote_path"; then
                                    failed_uploads=$((failed_uploads + 1))
                                fi
                            done
                            
                            cd ..
                            
                            if [ $failed_uploads -eq 0 ]; then
                                echo "✓ All files uploaded successfully!"
                            else
                                echo "⚠ $failed_uploads files failed to upload"
                                exit 1
                            fi
                            
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
                    
                    // Check if key files are accessible via FTP
                    withCredentials([string(credentialsId: 'infinityfree_ftp_pass', variable: 'FTP_PASSWORD')]) {
                        sh '''
                            echo "Checking if key files were deployed..."
                            
                            # Check if index.php exists in public directory
                            if curl -s --head "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}/public/index.php" > /dev/null 2>&1; then
                                echo "✓ public/index.php found on server"
                            else
                                echo "✗ public/index.php not found on server"
                            fi
                            
                            # Check if artisan file exists
                            if curl -s --head "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}/artisan" > /dev/null 2>&1; then
                                echo "✓ artisan file found on server"
                            else
                                echo "✗ artisan file not found on server"
                            fi
                            
                            # List some files to verify structure
                            echo "Listing root directory on server:"
                            curl -s "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}/" || echo "Could not list directory"
                        '''
                    }
                    
                    echo "Deployment verification completed!"
                }
            }
        }
    }
}
