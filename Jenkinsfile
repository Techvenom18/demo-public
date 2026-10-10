// Jenkinsfile (multi-language): GitHub -> Jenkins -> cPanel (SSH + rsync)
// Flow:   Checkout -> Build -> Test -> Deploy
// Layers: 1 Git history | 2 Backup before deploy | 4 Health check + rollback | 5 Safety rules
//
// The project type is detected from the files in the repo (first match wins):
//   composer.json              -> php     (composer, php -l, phpunit)
//   package.json               -> node    (npm ci, build, test)
//   CMakeLists.txt / Makefile  -> cpp     (compiled and tested only, NOT deployed)
//   any *.html file            -> static  (no build, uploaded as is)
//
// Needs on Jenkins:
//   Plugins:     NodeJS, SSH Agent, Lockable Resources
//   Tools:       NodeJS installation named 'node20' (Manage Jenkins -> Tools)
//   Credentials: SSH key with ID 'cpanel-ssh-key'
//   Job type:    Multibranch Pipeline (needed for  when { branch 'main' })
//   Packages:    rsync, openssh-client, plus php-cli + composer for PHP repos
//                and build-essential + cmake for C++ repos

pipeline {
    agent any

    tools { nodejs 'node20' }

    // Practice setup uses polling. On the company Jenkins use a GitHub webhook
    // and replace this line with:  githubPush()
    triggers { pollSCM('H/2 * * * *') }

    options {
        timeout(time: 20, unit: 'MINUTES')               // Layer 5: no endless builds
        buildDiscarder(logRotator(numToKeepStr: '20'))   // keep the last 20 builds only
    }

    environment {
        // ---- Change these per repo / per server ----
        APP_NAME    = 'demo'            // demo-private for the private repo
        HOST        = 'fakecpanel'      // real cPanel host later
        DEPLOY_USER = 'deploy'          // real cPanel SSH user later
        SSH_PORT    = '2222'            // real cPanel SSH port later
        NODE_MODE   = 'static'          // Node repos only:
                                        //   'static' = frontend, uploads dist/
                                        //   'server' = backend, uploads source, runs npm ci
                                        //              on the server and restarts the app
        // --------------------------------------------
        REMOTE_DIR  = "/config/public_html/${APP_NAME}"
        BACKUP_DIR  = "/config/backups/${APP_NAME}"
        SSH_OPTS    = "-p ${SSH_PORT} -o StrictHostKeyChecking=accept-new"
        LOCK_NAME   = "cpanel-${APP_NAME}"               // same name = deploys take turns
    }

    stages {

        // ---------------------------------------------------------------
        stage('Checkout') {
            steps {
                checkout scm

                script {
                    // Detect the project type
                    if (fileExists('composer.json')) {
                        env.PROJECT_TYPE = 'php'
                    } else if (fileExists('package.json')) {
                        env.PROJECT_TYPE = 'node'
                    } else if (fileExists('CMakeLists.txt') || fileExists('Makefile')) {
                        env.PROJECT_TYPE = 'cpp'
                    } else if (sh(script: 'find . -maxdepth 2 -name "*.html" -not -path "./node_modules/*" | head -n 1',
                                  returnStdout: true).trim()) {
                        env.PROJECT_TYPE = 'static'
                    } else {
                        error('Unknown project type: no composer.json, package.json, CMakeLists.txt, Makefile or .html file found')
                    }

                    // What to upload and how to check it afterwards
                    env.DEPLOY_ENABLED = 'true'
                    switch (env.PROJECT_TYPE) {
                        case 'php':
                            env.DEPLOY_SRC  = '.'
                            env.HEALTH_FILE = 'index.php'
                            break
                        case 'node':
                            if (env.NODE_MODE == 'server') {
                                env.DEPLOY_SRC  = '.'
                                env.HEALTH_FILE = 'package.json'
                            } else {
                                env.DEPLOY_SRC  = 'dist'
                                env.HEALTH_FILE = 'index.html'
                            }
                            break
                        case 'static':
                            env.DEPLOY_SRC  = '.'
                            env.HEALTH_FILE = 'index.html'
                            break
                        case 'cpp':
                            env.DEPLOY_ENABLED = 'false'   // compiled code is checked, not uploaded
                            env.DEPLOY_SRC  = ''
                            env.HEALTH_FILE = ''
                            break
                    }
                    echo "Detected project type: ${env.PROJECT_TYPE} | deploy enabled: ${env.DEPLOY_ENABLED}"
                }
            }
        }

        // ---------------------------------------------------------------
        stage('Build') {
            steps {
                script {
                    switch (env.PROJECT_TYPE) {
                        case 'php':
                            sh 'php -v && composer --version'
                            sh 'composer install --no-interaction --prefer-dist'
                            break
                        case 'node':
                            sh 'node -v && npm -v'
                            sh 'npm ci'
                            sh 'npm run build --if-present'
                            break
                        case 'cpp':
                            sh '''
                              if [ -f CMakeLists.txt ]; then
                                cmake -S . -B build && cmake --build build
                              else
                                make
                              fi
                            '''
                            break
                        case 'static':
                            echo 'Static HTML: no build step'
                            break
                    }
                }
            }
        }

        // ---------------------------------------------------------------
        // A failing test stops the pipeline here, before Deploy.
        stage('Test') {
            steps {
                script {
                    switch (env.PROJECT_TYPE) {
                        case 'php':
                            // Syntax check every PHP file outside vendor/
                            sh 'find . -name "*.php" -not -path "./vendor/*" -print0 | xargs -0 -r -n1 php -l'
                            // Unit tests, if the project has PHPUnit
                            sh '''
                              if [ -x vendor/bin/phpunit ]; then
                                vendor/bin/phpunit
                              else
                                echo "No PHPUnit found, skipping unit tests"
                              fi
                            '''
                            // Remove dev tools so only production code is uploaded
                            sh 'composer install --no-dev --optimize-autoloader --no-interaction'
                            break
                        case 'node':
                            sh 'npm test --if-present'
                            break
                        case 'cpp':
                            sh '''
                              if [ -f CMakeLists.txt ]; then
                                ctest --test-dir build --output-on-failure
                              elif make -n test > /dev/null 2>&1; then
                                make test
                              else
                                echo "No tests defined, skipping"
                              fi
                            '''
                            break
                        case 'static':
                            echo 'Static HTML: no automated tests'
                            break
                    }
                }
            }
        }

        // ---------------------------------------------------------------
        stage('Deploy') {
            when {
                allOf {
                    branch 'main'                                  // Layer 5: only main deploys
                    expression { env.DEPLOY_ENABLED == 'true' }    // C++ is never uploaded
                }
            }
            steps {
                // Optional: a newer build waiting at this point cancels an older waiting one
                milestone(1)

                // Layer 5: builds and tests run in parallel, but only one build
                // at a time may touch the server
                lock(resource: env.LOCK_NAME) {
                    sshagent(['cpanel-ssh-key']) {

                        // Files and folders that must never be uploaded or deleted on the server
                        writeFile file: '.rsync-exclude', text: [
                            '.git/', '.github/', '.gitignore', 'Jenkinsfile', '.rsync-exclude',
                            '.env', '.env.*', 'node_modules/', 'tests/', 'tmp/'
                        ].join('\n') + '\n'

                        // Layer 2: back up the live folder, keep the newest 5
                        sh '''
                          ssh $SSH_OPTS $DEPLOY_USER@$HOST "
                            mkdir -p $BACKUP_DIR $REMOTE_DIR &&
                            cp -a $REMOTE_DIR $BACKUP_DIR/backup-$BUILD_NUMBER &&
                            ls -1dt $BACKUP_DIR/backup-* | tail -n +6 | xargs -r rm -rf
                          "
                        '''

                        script {
                            try {
                                // Upload (excluded files are also protected from --delete)
                                sh '''
                                  rsync -az --delete \
                                    --exclude-from=.rsync-exclude \
                                    -e "ssh $SSH_OPTS" \
                                    $DEPLOY_SRC/ $DEPLOY_USER@$HOST:$REMOTE_DIR/
                                '''

                                // Node backend: install production packages and restart the app.
                                // The practice server has no Node, so this only works on the real host.
                                // On cPanel's "Setup Node.js App" you may also need to activate the
                                // app's virtual environment before npm (see the path in cPanel).
                                if (env.PROJECT_TYPE == 'node' && env.NODE_MODE == 'server') {
                                    sh '''
                                      ssh $SSH_OPTS $DEPLOY_USER@$HOST \
                                        "cd $REMOTE_DIR && npm ci --omit=dev && mkdir -p tmp && touch tmp/restart.txt"
                                    '''
                                }

                                // Layer 4: health check
                                sh 'ssh $SSH_OPTS $DEPLOY_USER@$HOST "test -f $REMOTE_DIR/nope.html"'
                                // On the real cPanel also check that the site responds:
                                // sh 'curl -fsS --max-time 20 https://YOUR-SUBDOMAIN/ > /dev/null'

                            } catch (err) {
                                // Layer 4: roll back inside the lock so no other deploy collides
                                echo 'Deploy failed: restoring the backup'
                                sh '''
                                  ssh $SSH_OPTS $DEPLOY_USER@$HOST \
                                    "rsync -a --delete $BACKUP_DIR/backup-$BUILD_NUMBER/ $REMOTE_DIR/"
                                '''
                                if (env.PROJECT_TYPE == 'node' && env.NODE_MODE == 'server') {
                                    sh 'ssh $SSH_OPTS $DEPLOY_USER@$HOST "mkdir -p $REMOTE_DIR/tmp && touch $REMOTE_DIR/tmp/restart.txt"'
                                }
                                throw err      // keep the build marked as failed
                            }
                        }
                    }
                }

                milestone(2)
            }
        }
    }

    post {
        success {
            script {
                if (env.DEPLOY_ENABLED == 'true') {
                    echo "Build ${env.BUILD_NUMBER} (${env.PROJECT_TYPE}) deployed and verified"
                } else {
                    echo "Build ${env.BUILD_NUMBER} (${env.PROJECT_TYPE}) built and tested. Nothing is deployed for this project type"
                }
            }
        }
        failure {
            // Replace with mail or Slack once your lead says who gets notified
            echo "ALERT: build ${env.BUILD_NUMBER} failed"
        }
        always { cleanWs() }
    }
}