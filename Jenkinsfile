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
                
                // Create production .env file
                sh '''
                    echo "Creating production environment file..."
                    cat > .env.production << 'EOF'
APP_NAME="Laravel Infinity App"
APP_ENV=production
APP_KEY=base64:your_app_key_here
APP_DEBUG=false
APP_URL=https://laravel-modular-kit.fwh.is

LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=error

DB_CONNECTION=sqlite
DB_DATABASE=/tmp/database.sqlite

BROADCAST_DRIVER=log
CACHE_DRIVER=file
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120

MEMCACHED_HOST=127.0.0.1

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=
AWS_USE_PATH_STYLE_ENDPOINT=false

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_HOST=
PUSHER_PORT=443
PUSHER_SCHEME=https
PUSHER_APP_CLUSTER=mt1

VITE_APP_NAME="${APP_NAME}"
VITE_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
VITE_PUSHER_HOST="${PUSHER_HOST}"
VITE_PUSHER_PORT="${PUSHER_PORT}"
VITE_PUSHER_SCHEME="${PUSHER_SCHEME}"
VITE_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
EOF
                '''
                
                // Create .htaccess for shared hosting (root level)
                sh '''
                    echo "Creating .htaccess for shared hosting..."
                    cat > .htaccess << 'EOF'
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews -Indexes
    </IfModule>

    RewriteEngine On

    # Handle Authorization Header
    RewriteCond %{HTTP:Authorization} .
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

    # Redirect Trailing Slashes If Not A Folder...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]

    # Send Requests To Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>

# Security headers
<IfModule mod_headers.c>
    Header always set X-Content-Type-Options nosniff
    Header always set X-Frame-Options DENY
    Header always set X-XSS-Protection "1; mode=block"
</IfModule>

# Disable directory browsing
Options -Indexes

# Disable server signature
ServerSignature Off

# Hide PHP version
<IfModule mod_headers.c>
    Header unset X-Powered-By
</IfModule>

# Error pages
ErrorDocument 404 /index.php
ErrorDocument 500 /index.php
EOF
                '''
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
                                   -not -path "./vendor/*" -not -path "./storage/logs/*" -not -name "*.log" -not -name ".env*" \
                                   -not -name "composer.lock" -not -name "package-lock.json" \
                                   -not -name ".gitignore" -not -name "README.md" -not -name "Jenkinsfile" \
                                   -exec cp --parents {} deploy_clean/ \\;
                            
                            # Copy the production .env file as .env
                            cp .env.production deploy_clean/.env
                            
                            # Copy public/index.php to root for shared hosting
                            echo "Setting up shared hosting structure..."
                            cp deploy_clean/public/index.php deploy_clean/index.php
                            
                            # Modify the root index.php to point to correct paths
                            sed -i 's|__DIR__."/\.\./vendor/autoload\.php"|__DIR__."/vendor/autoload.php"|g' deploy_clean/index.php
                            sed -i 's|__DIR__."/\.\./bootstrap/app\.php"|__DIR__."/bootstrap/app.php"|g' deploy_clean/index.php
                            
                            # Also copy all public assets to root
                            cp -r deploy_clean/public/* deploy_clean/ 2>/dev/null || true
                            
                            # Remove the duplicate index.php that was just copied
                            rm -f deploy_clean/public/index.php
                            
                            # Create necessary directories in deploy_clean
                            mkdir -p deploy_clean/storage/app/public
                            mkdir -p deploy_clean/storage/framework/cache/data
                            mkdir -p deploy_clean/storage/framework/sessions
                            mkdir -p deploy_clean/storage/framework/views
                            mkdir -p deploy_clean/storage/logs
                            mkdir -p deploy_clean/bootstrap/cache
                            
                            # Create empty index.html files to prevent directory browsing
                            echo '<html><head><title>403 Forbidden</title></head><body><h1>Directory access is forbidden.</h1></body></html>' > deploy_clean/storage/index.html
                            echo '<html><head><title>403 Forbidden</title></head><body><h1>Directory access is forbidden.</h1></body></html>' > deploy_clean/storage/app/index.html
                            echo '<html><head><title>403 Forbidden</title></head><body><h1>Directory access is forbidden.</h1></body></html>' > deploy_clean/storage/framework/index.html
                            echo '<html><head><title>403 Forbidden</title></head><body><h1>Directory access is forbidden.</h1></body></html>' > deploy_clean/bootstrap/index.html
                            
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
