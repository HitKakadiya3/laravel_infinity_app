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
                    // Upload everything to InfinityFree using FTP
                    publishOverFTP([
                        configName: 'InfinityFree',
                        transfers: [[
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
                }
            }
        }
    }
}
