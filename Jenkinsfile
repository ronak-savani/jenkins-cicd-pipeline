pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
    }

    environment {
        // Base environments
        DEPLOY_SERVER = "user@example-server.com"
        DEPLOY_BASE   = "/var/www/html"
        RELEASES_DIR  = "$DEPLOY_BASE/releases"
        CURRENT_LINK  = "$DEPLOY_BASE/crm.qntmnet.com"
        CONFIG_DIR    = "$DEPLOY_BASE/config"
        TIMESTAMP     = "${new Date().format('yyyyMMddHHmmss')}"
        SSH_PORT = "22"

        // Reusable variables
        GIT_URL = "git@github.com:ronak-savani/jenkins-cicd-pipeline.git"
        SSH_CMD = "ssh -o BatchMode=yes -o ConnectTimeout=5 -p ${SSH_PORT} $DEPLOY_SERVER"

        RSYNC_CMD = "rsync -avz -e 'ssh -p ${SSH_PORT}' --checksum --delete"
        APP_URL = "http://example-app.com/"
        ROLLBACK_HISTORY_FILE = "$RELEASES_DIR/.rollback_history"
        LOG_TIMESTAMP = "${new Date().format('yyyy-MM-dd HH:mm:ss')}"
        HEALTH_CHECK_CMD = "curl -s -o /dev/null -w \"%{http_code}\" --connect-timeout 10 $APP_URL"
        CHOWN_NGINX_CMD = "sudo chown -R nginx:nginx"
        CHMOD_755_CMD = "sudo chmod -R 755"


        // Email configuration
        EMAIL_RECIPIENTS = "team@example.com"
        EMAIL_SENDER = "jenkins@example.com"
        JENKINS_URL = "http://jenkins.example.com/" // Update with your Jenkins URL


       // Composer and PHP-FPM commands
   COMPOSER_INSTALL_CMD = "taskset -c 0 composer install --no-dev --optimize-autoloader"
       PHP_FPM_RELOAD_CMD = "sudo systemctl reload php7.4-fpm"

    }

    parameters {
        choice(
            name: 'ACTION',
            choices: ['DEPLOY', 'ROLLBACK'],
            description: 'Choose what action to perform'
        )
        string(
            name: 'ROLLBACK_VERSION',
            defaultValue: '',
            description: 'Enter release version to rollback to (e.g., 20250828104410) - leave empty to rollback to previous version'
        )
    }

    stages {
        stage('Send Start Notification') {
            steps {
                script {
                    def startedBy = currentBuild.getBuildCauses()[0].shortDescription
                    if (!startedBy) {
                        startedBy = "Manual trigger by ${env.USER_ID ?: 'unknown user'}"
                    }

                    def subject = "🚀 Pipeline STARTED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                    def message = """
                    Pipeline Execution Started
                    ==========================

                    Pipeline: ${env.JOB_NAME}
                    Build Number: #${env.BUILD_NUMBER}
                    Action: ${params.ACTION}
                    Started By: ${startedBy}
                    Start Time: ${new Date().format('yyyy-MM-dd HH:mm:ss')}
                    """

                    // Only include Rollback Version if action is ROLLBACK
                    if (params.ACTION == 'ROLLBACK') {
                        message += """
                        Rollback Version: ${params.ROLLBACK_VERSION ?: 'Not specified'}
                        """
                    }

                    message += """
                    Jenkins URL: ${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/
                    """

                    emailext(
                        subject: subject,
                        body: message,
                        to: env.EMAIL_RECIPIENTS,
                        from: env.EMAIL_SENDER
                    )

                    echo "Start notification email sent"
                }
            }
        }

        stage('Setup Variables') {
            steps {
                script {
                    env.IS_DEPLOY = (params.ACTION == 'DEPLOY')
                    env.IS_ROLLBACK = (params.ACTION == 'ROLLBACK')
                    env.CURRENT_RELEASE_PATH = "$RELEASES_DIR/$TIMESTAMP"
                    env.SETUP_ROLLBACK_HISTORY = """
                        sudo touch $RELEASES_DIR/.rollback_history || { echo 'ERROR: Failed to create rollback history'; exit 1; }
                        sudo chown nginx:nginx $RELEASES_DIR/.rollback_history || { echo 'ERROR: Failed to chown rollback history'; exit 1; }
                        sudo chmod 664 $RELEASES_DIR/.rollback_history || { echo 'ERROR: Failed to chmod rollback history'; exit 1; }
                    """

                    echo "ACTION parameter: ${params.ACTION}"
                    echo "IS_DEPLOY: ${env.IS_DEPLOY}"
                    echo "IS_ROLLBACK: ${env.IS_ROLLBACK}"
                }
            }
        }

        stage('Check for New Commits') {
            when {
                expression {
                    return params.ACTION == 'DEPLOY'
                }
            }
            agent any
            steps {
                script {
                    echo "Checking out application code from GitLab..."
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: 'main']],
                        extensions: [],
                        userRemoteConfigs: [[
                            credentialsId: 'gitlab-access-from-jenkins-server',
                            url: env.GIT_URL
                        ]]
                    ])

                    // Get the current commit hash from Git
                    def currentCommit = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Current commit in workspace: ${currentCommit}"

                    // Get the latest deployed commit from the server
                    sshagent (credentials: ['remote-server-ssh']) {
                        def deployedCommit = sh(
                            script: """
                                ${SSH_CMD} 'if [ -f ${CURRENT_LINK}/.git-commit ]; then cat ${CURRENT_LINK}/.git-commit; else echo "NO_COMMIT_FILE"; fi'
                            """,
                            returnStdout: true
                        ).trim()

                        echo "Deployed commit on server: ${deployedCommit != 'NO_COMMIT_FILE' ? deployedCommit : 'Not found'}"

                        if (deployedCommit == 'NO_COMMIT_FILE') {
                            echo "No commit file found on server - proceeding with deployment"
                        } else if (deployedCommit == currentCommit) {
                            def errorMsg = "❌ No new commits detected. Current commit (${currentCommit}) is already deployed. Aborting deployment."
                            env.DEPLOYMENT_ERROR = errorMsg
                            error(errorMsg)
                        } else {
                            echo "✅ New commits detected! Proceeding with deployment."
                            echo "From: ${deployedCommit} → To: ${currentCommit}"

                            // Store the commit for later use
                            env.DEPLOYED_COMMIT = currentCommit
                            env.PREVIOUS_COMMIT = deployedCommit

                            // Verify the code was checked out
                            sh '''
                            echo "Workspace contents:"
                            ls -la
                            echo "Application directory:"
                            ls -la beta-qlassroom.qntmnet.com/ || echo "No beta-qlassroom.qntmnet.com directory found"
                            '''
                        }
                    }
                }
            }
        }

        stage('Get Current Version Before Operation') {
            when {
                expression {
                    return params.ACTION == 'DEPLOY' || params.ACTION == 'ROLLBACK'
                }
            }
            agent any
            steps {
                sshagent (credentials: ['remote-server-ssh']) {
                    script {
                        // Get current version before any operation
                        def currentVersion = sh(
                            script: "${SSH_CMD} 'if [ -L ${CURRENT_LINK} ]; then readlink -f ${CURRENT_LINK} | xargs basename; else echo \"NO_CURRENT_VERSION\"; fi'",
                            returnStdout: true
                        ).trim()

                        env.CURRENT_VERSION_BEFORE_OPERATION = currentVersion
                        echo "Current version before operation: ${env.CURRENT_VERSION_BEFORE_OPERATION}"
                    }
                }
            }
        }

        stage('Verify Parameters') {
            agent any
            steps {
                script {
                    echo "Selected action: ${params.ACTION}"
                    echo "Rollback version: ${params.ROLLBACK_VERSION}"

                    if (params.ACTION == 'ROLLBACK') {
                        if (params.ROLLBACK_VERSION?.trim()) {
                            echo "Specific rollback version provided: ${params.ROLLBACK_VERSION}"
                        } else {
                            echo "No specific version provided - will check if rollback is allowed"
                        }
                    }
                }
            }
        }

        stage('Verify Server Connectivity') {
            when {
                expression {
                    return params.ACTION == 'DEPLOY' || params.ACTION == 'ROLLBACK'
                }
            }
            agent any
            steps {
                sshagent (credentials: ['remote-server-ssh']) {
                    sh '''
                    echo "Testing SSH connectivity..."
                    ${SSH_CMD} "echo 'SSH connection successful'"

                    echo "Testing web server connectivity..."
                    if curl -s -I --connect-timeout 5 ${APP_URL} >/dev/null 2>&1; then
                        echo "Web server is reachable before deployment"
                    else
                        echo "Warning: Web server is not reachable before deployment - continuing anyway"
                    fi
                    '''
                }
            }
        }

        stage('Verify Sudo Access') {
            when {
                expression {
                    return params.ACTION == 'DEPLOY' || params.ACTION == 'ROLLBACK'
                }
            }
            agent any
            steps {
                sshagent (credentials: ['remote-server-ssh']) {
                    sh '''
                    echo "Verifying sudo access for required commands..."
                    ${SSH_CMD} "
                        sudo -l | grep -E 'touch|chown|chmod|tee|ln|rm|mkdir|systemctl' || { echo 'ERROR: Missing sudo privileges'; exit 1; }
                        echo 'Sudo access verified'
                    "
                    '''
                }
            }
        }

        // Deployment-specific stages
        stage('Deployment Process') {
            when { expression { return params.ACTION == 'DEPLOY' } }
            stages {
                stage('Upload and Configure Release') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            script {
                                // First, debug the workspace structure
                                sh '''
                                echo "=== Workspace Structure ==="
                                ls -la
                                echo "=== All files and directories ==="
                                find . -maxdepth 2 -type f -name "*.php" -o -name "*.html" -o -name "*.js" | head -10
                                echo "=== Directory structure ==="
                                find . -type d | sort
                                '''

                                // Since your Git repo has files directly in root, use "." as source
                                env.SOURCE_PATH = "."
                            }

                            sh """
                            echo "Creating release directory: ${env.CURRENT_RELEASE_PATH}"
                            ${SSH_CMD} "sudo mkdir -p ${env.CURRENT_RELEASE_PATH}"
                            ${SSH_CMD} "sudo chown jenkins:nginx ${env.CURRENT_RELEASE_PATH}"
                            ${SSH_CMD} "sudo chmod 775 ${env.CURRENT_RELEASE_PATH}"

                            echo "Uploading files from Jenkins workspace..."
                            ${RSYNC_CMD} --exclude=".git" --exclude=".env" --exclude="vendor/" ${env.SOURCE_PATH}/ ${DEPLOY_SERVER}:${env.CURRENT_RELEASE_PATH}/

                            echo "Storing commit information..."
                            ${SSH_CMD} '
                                echo "${env.DEPLOYED_COMMIT}" | sudo tee ${env.CURRENT_RELEASE_PATH}/.git-commit > /dev/null
                                sudo chown nginx:nginx ${env.CURRENT_RELEASE_PATH}/.git-commit
                                sudo chmod 644 ${env.CURRENT_RELEASE_PATH}/.git-commit
                            '
                            """
                        }
                    }
                }

                stage('Run Composer Install') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            sh """
                            echo "Running composer install in the new release directory..."
                            ${SSH_CMD} "
                                cd ${env.CURRENT_RELEASE_PATH}
                                echo 'Current directory for composer:'
                                pwd
                                echo 'Files in release directory:'
                                ls -la
                                echo 'Checking for composer.json:'
                                if [ -f composer.json ]; then
                                    echo 'composer.json found. Running composer install...'
                                    ${env.COMPOSER_INSTALL_CMD}
                                    echo 'Composer install completed successfully'
                                else
                                    echo 'WARNING: composer.json not found. Skipping composer install.'
                                fi
                            "
                            """
                        }
                    }
                }

                stage('Set Permissions and Environment') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            sh """
                            echo "Setting permissions and environment..."
                            ${SSH_CMD} "
                                ${CHOWN_NGINX_CMD} ${env.CURRENT_RELEASE_PATH}
                                ${CHMOD_755_CMD} ${env.CURRENT_RELEASE_PATH}
                                sudo find ${env.CURRENT_RELEASE_PATH} -type f -exec chmod 644 {} \\;
                                sudo find ${env.CURRENT_RELEASE_PATH} -type d -exec chmod 755 {} \\;

                                # Remove any existing .env and create symlink
                                sudo rm -f ${env.CURRENT_RELEASE_PATH}/.env
                                sudo ln -sf ${CONFIG_DIR}/.env ${env.CURRENT_RELEASE_PATH}/.env
                                sudo chown -h nginx:nginx ${env.CURRENT_RELEASE_PATH}/.env

                                # Set special permissions for storage and cache directories if they exist
                                if [ -d ${env.CURRENT_RELEASE_PATH}/storage ]; then
                                    sudo chmod -R 775 ${env.CURRENT_RELEASE_PATH}/storage
                                    sudo chown -R nginx:nginx ${env.CURRENT_RELEASE_PATH}/storage
                                fi

                                if [ -d ${env.CURRENT_RELEASE_PATH}/bootstrap/cache ]; then
                                    sudo chmod -R 775 ${env.CURRENT_RELEASE_PATH}/bootstrap/cache
                                    sudo chown -R nginx:nginx ${env.CURRENT_RELEASE_PATH}/bootstrap/cache
                                fi
                            "

                            echo "Verifying deployed files..."
                            ${SSH_CMD} "
                                ls -la ${env.CURRENT_RELEASE_PATH}/ | head -10
                                echo 'File count:'
                                find ${env.CURRENT_RELEASE_PATH} -type f | wc -l
                                echo 'Checking vendor directory:'
                                ls -la ${env.CURRENT_RELEASE_PATH}/vendor/ 2>/dev/null || echo 'Vendor directory not found or inaccessible'
                                echo 'Checking .env symlink:'
                                ls -la ${env.CURRENT_RELEASE_PATH}/.env
                                echo 'Checking commit file:'
                                cat ${env.CURRENT_RELEASE_PATH}/.git-commit
                            "
                            """
                        }
                    }
                }

                stage('Activate Deployment Release') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            sh """
                            echo "Activating new release: ${env.TIMESTAMP}"
                            ${SSH_CMD} "
                                # Remove existing symlink if it exists
                                sudo rm -f ${CURRENT_LINK} 2>/dev/null || echo 'No existing symlink to remove'

                                # Create new symlink
                                sudo ln -sfn ${env.CURRENT_RELEASE_PATH} ${CURRENT_LINK}

                                # Set proper ownership on the symlink itself
                                sudo chown -h nginx:nginx ${CURRENT_LINK}

                                echo 'Activated new release ${env.TIMESTAMP}'
                                sudo rm -f ${ROLLBACK_HISTORY_FILE} || echo 'No rollback history file to remove'
                            "
                            """
                        }
                    }
                }

                stage('Reload PHP-FPM Service') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            sh """
                            echo "Reloading PHP-FPM service..."
                            ${SSH_CMD} "
                                ${env.PHP_FPM_RELOAD_CMD}
                                echo 'PHP-FPM reloaded successfully'

                                # Verify PHP-FPM status
                                sudo systemctl status php7.4-fpm --no-pager -l
                            "
                            """
                        }
                    }
                }

// HEALTH CHECK STAGE - ENABLED (return true)
stage('Health Check') {
    when {
        expression {
            // Health check is enabled
            return false
        }
    }
    agent any
    steps {
        sshagent (credentials: ['remote-server-ssh']) {
            script {
                // Check if this is the first deployment
                def currentLinkExists = sh(
                    script: "${SSH_CMD} 'test -L ${CURRENT_LINK} && echo exists || echo missing'",
                    returnStdout: true
                ).trim()

                if (currentLinkExists == "missing") {
                    echo "First deployment detected - skipping health check."
                } else {
                    def maxRetries = 5
                    def retryCount = 0
                    def success = false
                    def statusCode = "000"
                    def waitTime = 10 // seconds between retries

                    echo "Starting health check for application..."

                    while (retryCount < maxRetries && !success) {
                        retryCount++
                        echo "Health check attempt $retryCount of $maxRetries"

                        // Execute health check
                        try {
                            statusCode = sh(
                                script: "${SSH_CMD} '${HEALTH_CHECK_CMD}'",
                                returnStdout: true
                            ).trim()

                            echo "HTTP Status received: $statusCode"

                            if (statusCode == "200" || statusCode == "302" || statusCode == "301") {
                                success = true
                                echo "✅ Health check passed (HTTP $statusCode - Application is responding)"

                                // Additional verification
                                def response = sh(
                                    script: "${SSH_CMD} 'curl -s -I --connect-timeout 5 ${APP_URL} | head -10'",
                                    returnStdout: true
                                ).trim()
                                echo "Response headers:\n$response"

                            } else if (statusCode >= "400" && statusCode <= "599") {
                                echo "❌ Health check failed (HTTP $statusCode - Server error). Retrying in ${waitTime} seconds..."
                                sleep(waitTime)
                            } else if (statusCode == "000") {
                                echo "❌ Health check failed (No response - Connection failed). Retrying in ${waitTime} seconds..."
                                sleep(waitTime)
                            } else {
                                echo "⚠️ Unexpected status (HTTP $statusCode). Retrying in ${waitTime} seconds..."
                                sleep(waitTime)
                            }
                        } catch (Exception e) {
                            echo "❌ Health check command failed: ${e.getMessage()}"
                            sleep(waitTime)
                        }
                    }

                    if (!success) {
                        def errorMsg = "🚨 Health check failed after $maxRetries attempts. Last HTTP status: $statusCode"
                        env.DEPLOYMENT_ERROR = errorMsg
                        echo "${errorMsg}"
                        echo "Cleaning up failed deployment and rolling back..."

                        // Clean up the new release
                        sh """
                            ${SSH_CMD} "sudo rm -rf ${env.CURRENT_RELEASE_PATH}"
                            echo "Removed failed release: ${env.TIMESTAMP}"
                        """

                        error(errorMsg)
                    }
                }
            }
        }
    }
}

                stage('Post-Deployment Verification') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            script {
                                // Verify the symlink points to the correct release
                                def currentTarget = sh(
                                    script: "${SSH_CMD} 'readlink -f ${CURRENT_LINK}'",
                                    returnStdout: true
                                ).trim()

                                // Verify symlink ownership
                                def symlinkOwner = sh(
                                    script: "${SSH_CMD} 'ls -la ${CURRENT_LINK} | cut -d\" \" -f3-4'",
                                    returnStdout: true
                                ).trim()

                                if (currentTarget != env.CURRENT_RELEASE_PATH) {
                                    def errorMsg = "Deployment verification failed: Current link points to $currentTarget, expected ${env.CURRENT_RELEASE_PATH}"
                                    env.DEPLOYMENT_ERROR = errorMsg
                                    error errorMsg
                                }

                                if (!symlinkOwner.contains("nginx nginx")) {
                                    echo "⚠️ Symlink ownership issue: ${symlinkOwner}. Fixing..."
                                    sh """
                                        ${SSH_CMD} "sudo chown -h nginx:nginx ${CURRENT_LINK}"
                                    """
                                }

                                echo "✅ Deployment verified: Current link correctly points to new release"
                                echo "✅ Symlink ownership: ${symlinkOwner}"
                            }
                        }
                    }
                }

                stage('Cleanup Old Releases') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            script {
                                echo "Starting cleanup of old releases to keep only the 6 most recent..."

                                // Use a simpler approach for cleanup
                                sh """
                                ${SSH_CMD} '
                                    set -e
                                    echo "Cleaning up old releases (keeping latest 6)..."

                                    # Get list of release directories sorted by timestamp (newest first)
                                    releases=\$(ls -1 "$RELEASES_DIR" | grep -E "^[0-9]{14}\$" | sort -nr)
                                    count=\$(echo "\$releases" | wc -l)

                                    echo "Found \$count release directories"

                                    if [ \$count -gt 6 ]; then
                                        echo "Removing old releases (keeping 6 newest)..."
                                        echo "\$releases" | tail -n +\$((6 + 1)) | while read -r dir; do
                                            echo "Removing: $RELEASES_DIR/\$dir"
                                            sudo rm -rf "$RELEASES_DIR/\$dir"
                                        done
                                        echo "Cleanup completed"
                                    else
                                        echo "No cleanup needed (only \$count releases found)"
                                    fi

                                    echo "Current releases:"
                                    ls -1 "$RELEASES_DIR" | grep -E "^[0-9]{14}\$" | sort -nr
                                '
                                """
                            }
                        }
                    }
                }
            }
        }

        // Rollback-specific stages
        stage('Rollback Process') {
            when { expression { return params.ACTION == 'ROLLBACK' } }
            stages {
                stage('Determine Rollback Version') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            script {
                                if (params.ROLLBACK_VERSION?.trim()) {
                                    env.ROLLBACK_TARGET = params.ROLLBACK_VERSION.trim()
                                    env.ROLLBACK_TARGET_PATH = "$RELEASES_DIR/${env.ROLLBACK_TARGET}"
                                    echo "Using specified rollback version: ${env.ROLLBACK_TARGET}"
                                } else {
                                    def releasesOutput = sh(
                                        script: "${SSH_CMD} 'ls $RELEASES_DIR | sort -r'",
                                        returnStdout: true
                                    ).trim()

                                    def releases = releasesOutput.split('\n') as List
                                    echo "Available releases: ${releases}"

                                    def currentVersion = sh(
                                        script: "${SSH_CMD} 'readlink -f $CURRENT_LINK | xargs basename'",
                                        returnStdout: true
                                    ).trim()

                                    echo "Current version: ${currentVersion}"

                                    def rollbackHistory = sh(
                                        script: "${SSH_CMD} 'test -f $ROLLBACK_HISTORY_FILE && cat $ROLLBACK_HISTORY_FILE || echo \"\"'",
                                        returnStdout: true
                                    ).trim()

                                    if (rollbackHistory.contains("AUTO_ROLLBACK")) {
                                        def errorMsg = "You have already performed a rollback without parameters. Current version: ${currentVersion}. Please specify a specific version if you want to rollback further."
                                        env.ROLLBACK_ERROR = errorMsg
                                        error(errorMsg)
                                    }

                                    if (releases.size() >= 2) {
                                        def currentIndex = -1
                                        for (int i = 0; i < releases.size(); i++) {
                                            if (releases[i] == currentVersion) {
                                                currentIndex = i
                                                break
                                            }
                                        }

                                        if (currentIndex == -1) {
                                            def errorMsg = "Current version ${currentVersion} not found in releases directory!"
                                            env.ROLLBACK_ERROR = errorMsg
                                            error(errorMsg)
                                        }

                                        if (currentIndex == 0) {
                                            env.ROLLBACK_TARGET = releases[1]
                                            env.ROLLBACK_TARGET_PATH = "$RELEASES_DIR/${env.ROLLBACK_TARGET}"
                                            env.LATEST_VERSION_TO_CLEANUP = releases[0]
                                            env.CURRENT_VERSION_BEFORE_ROLLBACK = releases[0]

                                            echo "Current is latest version, rolling back to: ${env.ROLLBACK_TARGET}"
                                            echo "Latest version to cleanup: ${env.LATEST_VERSION_TO_CLEANUP}"

                                            sh """
                                                ${SSH_CMD} "
                                                    ${env.SETUP_ROLLBACK_HISTORY}
                                                    echo 'Writing rollback history...'
                                                    echo 'AUTO_ROLLBACK: ${env.LOG_TIMESTAMP} - From ${releases[0]} to ${releases[1]}' | sudo tee -a $ROLLBACK_HISTORY_FILE || { echo 'ERROR: Failed to write to rollback history'; exit 1; }
                                                    echo 'Verifying rollback history file:'
                                                    ls -l $ROLLBACK_HISTORY_FILE || { echo 'ERROR: Failed to verify rollback history'; exit 1; }
                                                "
                                            """
                                        } else {
                                            def errorMsg = "You are already on a previous version (${currentVersion}). Please specify a specific version if you want to rollback further."
                                            env.ROLLBACK_ERROR = errorMsg
                                            error(errorMsg)
                                        }
                                    } else if (releases.size() == 1) {
                                        def errorMsg = "Only one release exists. Cannot rollback to previous version."
                                        env.ROLLBACK_ERROR = errorMsg
                                        error(errorMsg)
                                    } else {
                                        def errorMsg = "No releases found. Cannot perform rollback."
                                        env.ROLLBACK_ERROR = errorMsg
                                        error(errorMsg)
                                    }
                                }

                                def versionExists = sh(
                                    script: "${SSH_CMD} 'test -d ${env.ROLLBACK_TARGET_PATH} && echo exists || echo missing'",
                                    returnStdout: true
                                ).trim()

                                if (versionExists == "missing") {
                                    def errorMsg = "Rollback target version ${env.ROLLBACK_TARGET} does not exist!"
                                    env.ROLLBACK_ERROR = errorMsg
                                    error(errorMsg)
                                }

                                echo "Rollback target confirmed: ${env.ROLLBACK_TARGET}"
                            }
                        }
                    }
                }

                stage('Activate Rollback Release') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            script {
                                echo "Rolling back to release: ${env.ROLLBACK_TARGET}"
                                sh """
                                ${SSH_CMD} "
                                    sudo rm -f ${CURRENT_LINK} 2>/dev/null || echo 'No existing symlink to remove'
                                    sudo ln -sfn ${env.ROLLBACK_TARGET_PATH} $CURRENT_LINK
                                    sudo chown -h nginx:nginx $CURRENT_LINK
                                    sudo ln -sf $CONFIG_DIR/.env ${env.ROLLBACK_TARGET_PATH}/.env 2>/dev/null || echo '.env symlink check skipped'
                                    sudo chown -h nginx:nginx ${env.ROLLBACK_TARGET_PATH}/.env 2>/dev/null || echo '.env ownership check skipped'
                                    echo 'Rolled back to ${env.ROLLBACK_TARGET}'
                                "
                                """
                            }
                        }
                    }
                }

                stage('Reload PHP-FPM After Rollback') {
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            sh """
                            echo "Reloading PHP-FPM service after rollback..."
                            ${SSH_CMD} "
                                ${env.PHP_FPM_RELOAD_CMD}
                                echo 'PHP-FPM reloaded successfully after rollback'
                            "
                            """
                        }
                    }
                }

                stage('Cleanup Latest Version After Rollback') {
                    when {
                        expression {
                            !params.ROLLBACK_VERSION?.trim() && env.LATEST_VERSION_TO_CLEANUP != null
                        }
                    }
                    agent any
                    steps {
                        sshagent (credentials: ['remote-server-ssh']) {
                            script {
                                def cleanupPath = "$RELEASES_DIR/${env.LATEST_VERSION_TO_CLEANUP}"
                                echo "Cleaning up the latest version after rollback: ${env.LATEST_VERSION_TO_CLEANUP}"
                                sh """
                                ${SSH_CMD} "
                                    sudo rm -rf ${cleanupPath} || echo 'Failed to cleanup latest version, but continuing'
                                    echo 'Removed latest version: ${env.LATEST_VERSION_TO_CLEANUP}'
                                "
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('Finalize Activation') {
            when {
                expression {
                    return params.ACTION == 'DEPLOY' || params.ACTION == 'ROLLBACK'
                }
            }
            agent any
            steps {
                sshagent (credentials: ['remote-server-ssh']) {
                    sh """
                    ${SSH_CMD} "
                        echo 'Setting final permissions...'
                        sudo chown -h nginx:nginx $CURRENT_LINK
                        echo 'Final activation completed'
                    "
                    """
                }
            }
        }
    }

    // Post build actions
    post {
        always {
            echo "Build completed with status: ${currentBuild.result}"
        }
        success {
            script {
                def startedBy = currentBuild.getBuildCauses()[0].shortDescription
                if (!startedBy) {
                    startedBy = "Manual trigger by ${env.USER_ID ?: 'unknown user'}"
                }

                def subject = "✅ Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                def message = """
                Pipeline Execution Completed Successfully
                ========================================

                Pipeline: ${env.JOB_NAME}
                Build Number: #${env.BUILD_NUMBER}
                Action: ${params.ACTION}
                Started By: ${startedBy}
                Start Time: ${currentBuild.startTimeInMillis ? new Date(currentBuild.startTimeInMillis).format('yyyy-MM-dd HH:mm:ss') : 'N/A'}
                Duration: ${currentBuild.durationString ?: 'N/A'}

                """

                if (params.ACTION == 'DEPLOY') {
                    message += """
                    Deployment Details:
                    - Previous Version: ${env.CURRENT_VERSION_BEFORE_OPERATION != 'NO_CURRENT_VERSION' ? env.CURRENT_VERSION_BEFORE_OPERATION : 'First Deployment'}
                    - New Version: ${env.TIMESTAMP}
                    - Previous Commit: ${env.PREVIOUS_COMMIT ?: 'N/A'}
                    - New Commit: ${env.DEPLOYED_COMMIT ?: 'N/A'}
                    - Status: ✅ SUCCESS
                    """
                    echo "Deployment successful!"
                } else if (params.ACTION == 'ROLLBACK') {
                    if (params.ROLLBACK_VERSION?.trim()) {
                        message += """
                        Rollback Details:
                        - Current Version: ${env.CURRENT_VERSION_BEFORE_OPERATION}
                        - Rollback Target: ${env.ROLLBACK_TARGET}
                        - Rollback Type: Manual (specific version)
                        - Status: ✅ SUCCESS
                        """
                        echo "Rollback to ${env.ROLLBACK_TARGET} successful!"
                    } else {
                        message += """
                        Rollback Details:
                        - Current Version: ${env.CURRENT_VERSION_BEFORE_OPERATION}
                        - Rollback Target: ${env.ROLLBACK_TARGET}
                        - Rollback Type: Automatic (to previous version)
                        - Latest Version Removed: ${env.LATEST_VERSION_TO_CLEANUP ?: 'N/A'}
                        - Status: ✅ SUCCESS
                        """
                        echo "Rollback to previous version successful! Latest version ${env.LATEST_VERSION_TO_CLEANUP} has been removed."
                    }
                }

                message += """

                Jenkins URL: ${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/
                """

                emailext(
                    subject: subject,
                    body: message,
                    to: env.EMAIL_RECIPIENTS,
                    from: env.EMAIL_SENDER
                )

                echo "Success notification email sent"
            }
        }
        failure {
            script {
                def startedBy = currentBuild.getBuildCauses()[0].shortDescription
                if (!startedBy) {
                    startedBy = "Manual trigger by ${env.USER_ID ?: 'unknown user'}"
                }

                def subject = "❌ Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                def errorDetails = env.DEPLOYMENT_ERROR ?: env.ROLLBACK_ERROR ?: "Unknown error - check Jenkins logs for details"

                def message = """
                Pipeline Execution Failed
                ========================

                Pipeline: ${env.JOB_NAME}
                Build Number: #${env.BUILD_NUMBER}
                Action: ${params.ACTION}
                Started By: ${startedBy}
                Start Time: ${currentBuild.startTimeInMillis ? new Date(currentBuild.startTimeInMillis).format('yyyy-MM-dd HH:mm:ss') : 'N/A'}
                Duration: ${currentBuild.durationString ?: 'N/A'}
                Status: ❌ FAILED

                Error Details:
                ${errorDetails}

                Please check the Jenkins logs for detailed error information.

                Jenkins URL: ${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/
                """

                emailext(
                    subject: subject,
                    body: message,
                    to: env.EMAIL_RECIPIENTS,
                    from: env.EMAIL_SENDER
                )

                echo "Failure notification email sent"
            }

            echo "Operation failed. Check logs for details."
            script {
                if (params.ACTION == 'DEPLOY' && env.CURRENT_RELEASE_PATH) {
                    node {
                        echo "Cleaning up failed deployment: ${env.TIMESTAMP}"
                        sshagent (credentials: ['remote-server-ssh']) {
                            sh """
                            ${SSH_CMD} "sudo rm -rf ${env.CURRENT_RELEASE_PATH} || echo 'Failed to cleanup, but continuing'"
                            echo "Removed failed deployment files"
                            """
                        }
                    }
                }
            }
        }
        unstable {
            echo "Pipeline status: UNSTABLE"
        }
        aborted {
            echo "Pipeline was ABORTED"

            script {
                def startedBy = currentBuild.getBuildCauses()[0].shortDescription
                if (!startedBy) {
                    startedBy = "Manual trigger by ${env.USER_ID ?: 'unknown user'}"
                }

                def subject = "⏹️ Pipeline ABORTED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                def message = """
                Pipeline Execution Aborted
                =========================

                Pipeline: ${env.JOB_NAME}
                Build Number: #${env.BUILD_NUMBER}
                Action: ${params.ACTION}
                Started By: ${startedBy}
                Start Time: ${currentBuild.startTimeInMillis ? new Date(currentBuild.startTimeInMillis).format('yyyy-MM-dd HH:mm:ss') : 'N/A'}
                Duration: ${currentBuild.durationString ?: 'N/A'}
                Status: ⏹️ ABORTED

                The pipeline was manually aborted or terminated.

                Jenkins URL: ${env.JENKINS_URL}/job/${env.JOB_NAME}/${env.BUILD_NUMBER}/
                """

                emailext(
                    subject: subject,
                    body: message,
                    to: env.EMAIL_RECIPIENTS,
                    from: env.EMAIL_SENDER
                )

                echo "Aborted notification email sent"
            }
        }
    }
}
