pipeline {
    agent any

    environment {
        FTP_HOST = 'ftpupload.net'
        FTP_USER = 'if0_40134274'
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
                
                // Generate APP_KEY first
                sh '''
                    echo "Generating application key..."
                    php artisan key:generate --show > app_key.txt
                    APP_KEY_GENERATED=$(cat app_key.txt)
                    echo "Generated key: $APP_KEY_GENERATED"
                '''
                
                sh 'php artisan config:cache'
                sh 'php artisan route:cache'
                sh 'php artisan view:cache'
                
                // Create production .env file with generated key
                sh '''
                    APP_KEY_GENERATED=$(cat app_key.txt)
                    echo "Creating production environment file with key: $APP_KEY_GENERATED"
                    cat > .env.production << EOF
APP_NAME="Laravel Infinity App"
APP_ENV=production
APP_KEY=$APP_KEY_GENERATED
APP_DEBUG=false
APP_URL=https://laravel-modular-kit.fwh.is

LOG_CHANNEL=single
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=error
LOG_SINGLE_TARGET=/tmp/laravel.log

DB_CONNECTION=sqlite
DB_DATABASE=/tmp/database.sqlite

BROADCAST_DRIVER=log
CACHE_DRIVER=file
CACHE_PATH=/tmp/cache
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120
SESSION_PATH=/tmp

MEMCACHED_HOST=127.0.0.1

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=log
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

VIEW_COMPILED_PATH=/tmp/views
CACHE_COMPILED_PATH=/tmp/cache
SESSION_COMPILED_PATH=/tmp/sessions
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
                            
                            # Copy all project files EXCEPT specified exclusions
                            echo "Copying all files for deployment..."
                            rsync -av --exclude='.git' --exclude='node_modules' --exclude='tests' \
                                  --exclude='storage/logs/*' --exclude='*.log' --exclude='.env*' \
                                  --exclude='composer.lock' --exclude='package-lock.json' \
                                  --exclude='.gitignore' --exclude='README.md' --exclude='Jenkinsfile' \
                                  --exclude='app_key.txt' --exclude='vendor' \
                                  ./ deploy_clean/
                            
                            # Note: Vendor directory excluded - will be uploaded manually
                            echo "Vendor directory excluded from automated deployment"
                            echo "Note: Upload vendor directory manually to the server"
                            
                            # Copy the production .env file as .env
                            cp .env.production deploy_clean/.env
                            
                            # Create necessary directories in /tmp (allowed path)
                            mkdir -p deploy_clean/storage/app/public
                            mkdir -p deploy_clean/storage/framework/cache/data
                            mkdir -p deploy_clean/storage/framework/sessions
                            mkdir -p deploy_clean/storage/framework/views
                            mkdir -p deploy_clean/storage/logs
                            mkdir -p deploy_clean/bootstrap/cache
                            mkdir -p deploy_clean/database
                            
                            # Create SQLite database file in /tmp (allowed path)
                            touch deploy_clean/database/database.sqlite
                            chmod 666 deploy_clean/database/database.sqlite
                            
                            # Create a custom logging configuration to handle open_basedir restrictions
                            cat > deploy_clean/config/logging.php << 'PHPEOF'
<?php

use Monolog\Handler\NullHandler;
use Monolog\Handler\StreamHandler;
use Monolog\Handler\SyslogUdpHandler;

return [

    /*
    |--------------------------------------------------------------------------
    | Default Log Channel
    |--------------------------------------------------------------------------
    |
    | This option defines the default log channel that gets used when writing
    | messages to the logs. The name specified in this option should match
    | one of the channels defined in the "channels" configuration array.
    |
    */

    'default' => env('LOG_CHANNEL', 'single'),

    /*
    |--------------------------------------------------------------------------
    | Deprecations Log Channel
    |--------------------------------------------------------------------------
    |
    | This option controls the log channel that should be used to log warnings
    | regarding deprecated PHP and library features. This allows you to get
    | your application ready for upcoming major versions of dependencies.
    |
    */

    'deprecations' => [
        'channel' => env('LOG_DEPRECATIONS_CHANNEL', 'null'),
        'trace' => false,
    ],

    /*
    |--------------------------------------------------------------------------
    | Log Channels
    |--------------------------------------------------------------------------
    |
    | Here you may configure the log channels for your application. Out of
    | the box, Laravel uses the Monolog PHP logging library. This gives
    | you a variety of powerful log handlers / formatters to utilize.
    |
    | Available Drivers: "single", "daily", "slack", "syslog",
    |                    "errorlog", "monolog",
    |                    "custom", "stack"
    |
    */

    'channels' => [
        'stack' => [
            'driver' => 'stack',
            'channels' => ['single'],
            'ignore_exceptions' => false,
        ],

        'single' => [
            'driver' => 'single',
            'path' => env('LOG_SINGLE_TARGET', '/tmp/laravel.log'),
            'level' => env('LOG_LEVEL', 'debug'),
        ],

        'daily' => [
            'driver' => 'daily',
            'path' => '/tmp/laravel.log',
            'level' => env('LOG_LEVEL', 'debug'),
            'days' => 14,
        ],

        'slack' => [
            'driver' => 'slack',
            'url' => env('LOG_SLACK_WEBHOOK_URL'),
            'username' => 'Laravel Log',
            'emoji' => ':boom:',
            'level' => env('LOG_LEVEL', 'critical'),
        ],

        'papertrail' => [
            'driver' => 'monolog',
            'level' => env('LOG_LEVEL', 'debug'),
            'handler' => env('LOG_PAPERTRAIL_HANDLER', SyslogUdpHandler::class),
            'handler_with' => [
                'host' => env('PAPERTRAIL_URL'),
                'port' => env('PAPERTRAIL_PORT'),
                'connectionString' => 'tls://'.env('PAPERTRAIL_URL').':'.env('PAPERTRAIL_PORT'),
            ],
        ],

        'stderr' => [
            'driver' => 'monolog',
            'level' => env('LOG_LEVEL', 'debug'),
            'handler' => StreamHandler::class,
            'formatter' => env('LOG_STDERR_FORMATTER'),
            'with' => [
                'stream' => 'php://stderr',
            ],
        ],

        'syslog' => [
            'driver' => 'syslog',
            'level' => env('LOG_LEVEL', 'debug'),
        ],

        'errorlog' => [
            'driver' => 'errorlog',
            'level' => env('LOG_LEVEL', 'debug'),
        ],

        'null' => [
            'driver' => 'monolog',
            'handler' => NullHandler::class,
        ],

        'emergency' => [
            'path' => '/tmp/laravel.log',
        ],
    ],

];
PHPEOF
                            
                            # Copy public/index.php to root and modify for shared hosting
                            echo "Setting up shared hosting structure..."
                            if [ -f deploy_clean/public/index.php ]; then
                                cp deploy_clean/public/index.php deploy_clean/index.php
                                
                                # Create a properly modified index.php for shared hosting with better error handling
                                cat > deploy_clean/index.php << 'PHPEOF'
<?php
// Enable error reporting for debugging
error_reporting(E_ALL);
ini_set('display_errors', 1);

use Illuminate\\Contracts\\Http\\Kernel;
use Illuminate\\Http\\Request;

define('LARAVEL_START', microtime(true));

// Debug: Check if files exist
if (!file_exists(__DIR__.'/vendor/autoload.php')) {
    die('Error: vendor/autoload.php not found. Path: ' . __DIR__ . '/vendor/autoload.php');
}

if (!file_exists(__DIR__.'/bootstrap/app.php')) {
    die('Error: bootstrap/app.php not found. Path: ' . __DIR__ . '/bootstrap/app.php');
}

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

try {
    require __DIR__.'/vendor/autoload.php';
} catch (Exception $e) {
    die('Error loading autoload: ' . $e->getMessage());
}

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

try {
    $app = require_once __DIR__.'/bootstrap/app.php';

    $kernel = $app->make(Kernel::class);

    $response = $kernel->handle(
        $request = Request::capture()
    )->send();

    $kernel->terminate($request, $response);
} catch (Exception $e) {
    echo '<h1>Laravel Application Error</h1>';
    echo '<p><strong>Error:</strong> ' . htmlspecialchars($e->getMessage()) . '</p>';
    echo '<p><strong>File:</strong> ' . htmlspecialchars($e->getFile()) . '</p>';
    echo '<p><strong>Line:</strong> ' . $e->getLine() . '</p>';
    echo '<p><a href="/debug.php">View Debug Information</a></p>';
    echo '<pre>' . htmlspecialchars($e->getTraceAsString()) . '</pre>';
}
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
                            
                            echo "Files prepared for deployment:"
                            echo "Total files: $(find deploy_clean -type f | wc -l)"
                            echo "Vendor directory size: $(du -sh deploy_clean/vendor 2>/dev/null || echo 'Not found')"
                            echo "Key files check:"
                            ls -la deploy_clean/index.php deploy_clean/.env deploy_clean/artisan deploy_clean/vendor/autoload.php 2>/dev/null || true
                            
                            # Upload all files (only non-empty ones)
                            cd deploy_clean
                            failed_uploads=0
                            total_files=$(find . -type f -not -empty | wc -l)
                            current_file=0
                            
                            echo "Uploading $total_files non-empty files..."
                            
                            # Upload critical files first
                            priority_files=".env index.php debug.php artisan"
                            
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
                            
                            echo "Uploading remaining files (excluding vendor)..."
                            find . -type f -not -empty -not -path "./vendor/*" | while read file; do
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
                                    echo "[$current_file/$total_files] Regular: $remote_path"
                                    if ! upload_file "$file" "/$remote_path"; then
                                        failed_uploads=$((failed_uploads + 1))
                                    fi
                                fi
                            done
                            
                            echo "=== Deployment Summary ==="
                            echo "Total files processed: $total_files"
                            echo "Failed uploads: $failed_uploads"
                            echo "⚠ Remember to manually upload the vendor directory!"
                            

                            if [ $failed_uploads -gt 0 ]; then
                                echo "⚠ Warning: $failed_uploads files failed to upload"
                                echo "Deployment completed with warnings"
                            else
                                echo "✓ All files uploaded successfully!"
                            fi
                            
                            echo "Deployment to InfinityFree completed!"
                            echo "Site should be available at: https://laravel-modular-kit.fwh.is"
                            echo "Debug info available at: https://laravel-modular-kit.fwh.is/debug.php"
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'Deployment completed successfully!'
        }
        failure {
            echo 'Deployment failed. Check the logs for details.'
        }
    }
}
