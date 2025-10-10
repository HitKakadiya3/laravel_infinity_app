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
DB_DATABASE=database/database.sqlite

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
                                   -not -empty -exec cp --parents {} deploy_clean/ \\;
                            
                            # Copy the production .env file as .env
                            cp .env.production deploy_clean/.env
                            
                            # Create necessary directories
                            mkdir -p deploy_clean/storage/app/public
                            mkdir -p deploy_clean/storage/framework/cache/data
                            mkdir -p deploy_clean/storage/framework/sessions
                            mkdir -p deploy_clean/storage/framework/views
                            mkdir -p deploy_clean/storage/logs
                            mkdir -p deploy_clean/bootstrap/cache
                            mkdir -p deploy_clean/database
                            
                            # Create SQLite database file
                            touch deploy_clean/database/database.sqlite
                            
                            # Copy public/index.php to root and modify for shared hosting
                            echo "Setting up shared hosting structure..."
                            if [ -f deploy_clean/public/index.php ]; then
                                cp deploy_clean/public/index.php deploy_clean/index.php
                                
                                # Create a properly modified index.php for shared hosting
                                cat > deploy_clean/index.php << 'PHPEOF'
<?php

use Illuminate\\Contracts\\Http\\Kernel;
use Illuminate\\Http\\Request;

define('LARAVEL_START', microtime(true));

/*
|--------------------------------------------------------------------------
| Check If The Application Is Under Maintenance
|--------------------------------------------------------------------------
|
| If the application is in maintenance / demo mode via the "down" command
| we will load this file so that any pre-rendered content can be shown
| instead of starting the framework, which could cause an exception.
|
*/

if (file_exists($maintenance = __DIR__.'/storage/framework/maintenance.php')) {
    require $maintenance;
}

/*
|--------------------------------------------------------------------------
| Register The Auto Loader
|--------------------------------------------------------------------------
|
| Composer provides a convenient, automatically generated class loader for
| this application. We just need to utilize it! We'll simply require it
| into the script here so we don't need to manually load our classes.
|
*/

require __DIR__.'/vendor/autoload.php';

/*
|--------------------------------------------------------------------------
| Run The Application
|--------------------------------------------------------------------------
|
| Once we have the application, we can handle the incoming request using
| the application's HTTP kernel. Then, we will send the response back
| to this client's browser, allowing them to enjoy our application.
|
*/

$app = require_once __DIR__.'/bootstrap/app.php';

$kernel = $app->make(Kernel::class);

$response = $kernel->handle(
    $request = Request::capture()
)->send();

$kernel->terminate($request, $response);
PHPEOF
                            fi
                            
                            # Copy other public assets to root (excluding index.php)
                            echo "Copying public assets..."
                            if [ -d deploy_clean/public ]; then
                                find deploy_clean/public -type f -not -name "index.php" -not -empty -exec sh -c '
                                    for file do
                                        filename=$(basename "$file")
                                        if [ -s "$file" ]; then
                                            cp "$file" deploy_clean/
                                        fi
                                    done
                                ' sh {} +
                            fi
                            
                            echo "Files prepared for deployment:"
                            find deploy_clean -type f | head -10
                            echo "... and $(find deploy_clean -type f | wc -l) total files"
                            
                            echo "Uploading files using curl..."
                            
                            # Function to upload a single file
                            upload_file() {
                                local file="$1"
                                local remote_path="$2"
                                local remote_dir=$(dirname "$remote_path")
                                
                                # Check if file exists and is not empty
                                if [ ! -f "$file" ] || [ ! -s "$file" ]; then
                                    echo "⚠ Skipping empty or non-existent file: $remote_path"
                                    return 0
                                fi
                                
                                # Create directory if it doesn't exist
                                curl -s --ftp-create-dirs -T /dev/null "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}${remote_dir}/" || true
                                
                                # Upload the file with retry
                                local max_retries=3
                                local retry=0
                                
                                while [ $retry -lt $max_retries ]; do
                                    if curl -T "$file" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}${remote_path}"; then
                                        echo "✓ Uploaded: $remote_path ($(stat -c%s "$file") bytes)"
                                        
                                        # Set proper permissions for PHP files
                                        if [[ "$remote_path" == *.php ]] || [[ "$remote_path" == */artisan ]]; then
                                            curl -Q "SITE CHMOD 755 ${FTP_PATH}${remote_path}" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                                        fi
                                        
                                        return 0
                                    else
                                        retry=$((retry + 1))
                                        echo "⚠ Retry $retry/$max_retries for: $remote_path"
                                        sleep 2
                                    fi
                                done
                                
                                echo "✗ Failed after $max_retries attempts: $remote_path"
                                return 1
                            }
                            
                            # Upload all files (only non-empty ones)
                            cd deploy_clean
                            failed_uploads=0
                            total_files=$(find . -type f -not -empty | wc -l)
                            current_file=0
                            
                            echo "Uploading $total_files non-empty files..."
                            
                            # Upload files with priority order
                            priority_files=".env index.php artisan"
                            
                            echo "Uploading priority files first..."
                            for pfile in $priority_files; do
                                if [ -f "$pfile" ] && [ -s "$pfile" ]; then
                                    current_file=$((current_file + 1))
                                    echo "[$current_file/$total_files] Priority: $pfile"
                                    if ! upload_file "$pfile" "/$pfile"; then
                                        failed_uploads=$((failed_uploads + 1))
                                    fi
                                fi
                            done
                            
                            echo "Uploading remaining files..."
                            find . -type f -not -empty | while read file; do
                                remote_path="${file#./}"
                                
                                # Skip if already uploaded as priority
                                skip=false
                                for pfile in $priority_files; do
                                    if [ "$remote_path" = "$pfile" ]; then
                                        skip=true
                                        break
                                    fi
                                done
                                
                                if [ "$skip" = false ]; then
                                    current_file=$((current_file + 1))
                                    echo "[$current_file/$total_files] Processing: $remote_path"
                                    
                                    if ! upload_file "$file" "/$remote_path"; then
                                        failed_uploads=$((failed_uploads + 1))
                                    fi
                                fi
                            done
                            
                            cd ..
                            
                            if [ $failed_uploads -eq 0 ]; then
                                echo "✓ All files uploaded successfully!"
                            else
                                echo "⚠ $failed_uploads files failed to upload"
                                exit 1
                            fi
                            
                            # Set directory permissions
                            echo "Setting directory permissions..."
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/storage" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/bootstrap" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/bootstrap/cache" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/storage/framework" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/storage/framework/cache" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/storage/framework/sessions" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/storage/framework/views" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/storage/logs" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 755 ${FTP_PATH}/database" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            curl -Q "SITE CHMOD 644 ${FTP_PATH}/database/database.sqlite" "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}/" || true
                            
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
                            
                            # Check if index.php exists in root
                            if curl -s --head "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}/index.php" > /dev/null 2>&1; then
                                echo "✓ index.php found in root"
                            else
                                echo "✗ index.php not found in root"
                            fi
                            
                            # Check if .env file exists
                            if curl -s --head "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}/.env" > /dev/null 2>&1; then
                                echo "✓ .env file found on server"
                            else
                                echo "✗ .env file not found on server"
                            fi
                            
                            # Check if artisan file exists
                            if curl -s --head "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}/artisan" > /dev/null 2>&1; then
                                echo "✓ artisan file found on server"
                            else
                                echo "✗ artisan file not found on server"
                            fi
                            
                            # Check if vendor directory exists
                            if curl -s --head "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}/vendor/" > /dev/null 2>&1; then
                                echo "✓ vendor directory found on server"
                            else
                                echo "✗ vendor directory not found on server"
                            fi
                            
                            # List some files to verify structure
                            echo "Listing root directory on server:"
                            curl -s "ftp://${FTP_USER}:${FTP_PASSWORD}@${FTP_HOST}${FTP_PATH}/" | head -20 || echo "Could not list directory"
                        '''
                    }
                    
                    // Try to check the website response
                    sh '''
                        echo "Checking website response..."
                        response=$(curl -s -o /dev/null -w "%{http_code}" https://laravel-modular-kit.fwh.is/ || echo "000")
                        echo "Website response code: $response"
                        
                        if [ "$response" = "200" ]; then
                            echo "✓ Website is responding successfully!"
                        elif [ "$response" = "500" ]; then
                            echo "⚠ Website still returning 500 error - check server logs"
                        else
                            echo "⚠ Unexpected response code: $response"
                        fi
                    '''
                    
                    echo "Deployment verification completed!"
                }
            }
        }
    }
}
