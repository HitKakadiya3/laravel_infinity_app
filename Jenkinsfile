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
                sh 'php artisan config:cache'
                sh 'php artisan route:cache'
                sh 'php artisan view:cache'
            }
        }

        stage('Deploy to InfinityFree') {
            steps {
                script {
                    // Try using step wrapper for FTP publish
                    try {
                        step([
                            $class: 'jenkins.plugins.publish_over_ftp.BapFtpPublisher',
                            configName: 'InfinityFree',
                            transfers: [[
                                $class: 'jenkins.plugins.publish_over_ftp.BapFtpTransfer',
                                asciiMode: false,
                                cleanRemote: false,
                                excludes: '.git/**, .gitignore, composer.lock, package-lock.json, node_modules/**, tests/**, storage/logs/**',
                                flatten: false,
                                makeEmptyDirs: false,
                                noDefaultExcludes: false,
                                patternSeparator: '[, ]+',
                                remoteDirectory: "${FTP_PATH}",
                                remoteDirectorySDF: false,
                                removePrefix: '',
                                sourceFiles: '**/*'
                            ]],
                            usePromotionTimestamp: false,
                            verbose: true
                        ])
                    } catch (Exception e) {
                        echo "FTP plugin not available, using alternative method..."
                        
                        // Fallback: Create a ZIP file and provide instructions
                        if (isUnix()) {
                            sh '''
                                # Create deployment package
                                zip -r laravel_app_deploy.zip . -x "*.git*" "node_modules/*" "tests/*" "storage/logs/*" "*.log" "composer.lock" "package-lock.json"
                                echo "Deployment package created: laravel_app_deploy.zip"
                                echo "Please manually upload this file to your FTP server"
                            '''
                        } else {
                            powershell '''
                                # Create deployment package using PowerShell
                                $excludePatterns = @("*.git*", "node_modules", "tests", "storage\\logs", "*.log", "composer.lock", "package-lock.json")
                                $filesToZip = Get-ChildItem -Recurse -File | Where-Object {
                                    $file = $_
                                    $shouldExclude = $false
                                    foreach ($pattern in $excludePatterns) {
                                        if ($file.FullName -like "*$pattern*") {
                                            $shouldExclude = $true
                                            break
                                        }
                                    }
                                    -not $shouldExclude
                                }
                                
                                Compress-Archive -Path $filesToZip.FullName -DestinationPath "laravel_app_deploy.zip" -Force
                                Write-Host "Deployment package created: laravel_app_deploy.zip"
                                Write-Host "Please manually upload this file to your FTP server"
                            '''
                        }
                        
                        // Archive the deployment package for download
                        archiveArtifacts artifacts: 'laravel_app_deploy.zip', fingerprint: true
                    }
                }
            }
        }
    }
}
