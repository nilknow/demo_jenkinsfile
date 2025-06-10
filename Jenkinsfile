pipeline {
    agent any // Jenkins agent (must be a Windows machine with Docker Desktop, configured for Windows Containers)

    parameters {
        string(name: 'PARALLEL_COUNT', defaultValue: '3', description: 'Number of parallel Windows containers to run tests in')
        string(name: 'WINDOWS_VERSION', defaultValue: '2025', description: 'Windows version for dockur/windows (e.g., 10, 11, 2025)')
        string(name: 'TEST_SCRIPT', defaultValue: 'powershell.exe -File C:\\project\\run_tests.ps1', description: 'PowerShell command to execute tests inside the container')
    }

    options {
        // Timeout for the entire pipeline
        timeout(time: 240, unit: 'MINUTES')
        skipDefaultCheckout true
    }

    environment {
        DOCKUR_IMAGE = "ghcr.io/dockur/windows"
        PROJECT_DIR_IN_CONTAINER = "C:\\Users\\Docker\\project" // Windows path inside the container
        REPORT_DIR_IN_CONTAINER = "C:\\Users\\Docker\\test-results" // Windows path for reports
    }

    stages {
        stage('Prepare Workspace and Stash') {
            steps {
                script {
                    echo "Checking out SCM..."

                    def repoUrl = 'https://github.com/nilknow/temp.git'
                    sh "mkdir source/"
                    sh "git clone ${repoUrl}"

                    // Ensure the target directory for test results exists on the agent
                    sh "mkdir -p target/test-results"

                    echo "Stashing project source for parallel stages..."
                    // Stash the workspace content to be available for parallel stages
                    // Windows Docker containers often require paths in Windows format for volume mounts
                    // Adjust 'includes' if your project has a very specific structure
                    stash includes: '**/*', name: 'project-source'
                }
            }
        }

        stage('Run Parallel Windows Tests') {
            steps {
                script {
                    def parallelStages = [:]
                    def testCommands = []

                    def parallelCountInt = PARALLEL_COUNT.toInteger()

                    for (int i = 0; i < parallelCountInt; i++) {
                        testCommands.add(env.TEST_SCRIPT)
                    }

                    echo "start parallel stages, ${PARALLEL_COUNT}, ${WINDOWS_VERSION}, ${TEST_SCRIPT}"

                    testCommands.eachWithIndex { cmd, i ->
                        def stageName = "Windows Test Instance ${i + 1}"
                        def containerName = "windows-test-${env.BUILD_NUMBER}-${i}"
                        def agentReportDir = "${pwd()}/target/test-results/instance-${i}" // Unique dir on agent

                        echo "run ${stageName} in container ${containerName}"

                        parallelStages[stageName] = {
                            // No 'steps' block here, as we are already inside a Groovy closure for dynamic parallelism
                            script { // This `script` block is necessary to run Groovy logic
                                // Create a directory for this instance's results on the Jenkins agent
                                sh "mkdir -p ${agentReportDir}"

                                echo "Starting ${DOCKUR_IMAGE}:${WINDOWS_VERSION} container named ${containerName}..."
                                sh """
                                    docker run -d \\
                                        -e VERSION=${WINDOWS_VERSION} \\
                                        -e DISK_SIZE=32G \\
                                        --name ${containerName} \\
                                        -v "${pwd()}:${PROJECT_DIR_IN_CONTAINER}" \\
                                        -v "${agentReportDir}:${REPORT_DIR_IN_CONTAINER}" \\
                                        ${DOCKUR_IMAGE}
                                """
                                sh "sleep 120" // Give Windows VM time to boot

                                echo "Executing tests in ${containerName}..."
                                try {
                                    unstash 'project-source'

                                    sh """
                                        docker exec ${containerName} ${TEST_SCRIPT}
                                    """

                                } catch (Exception e) {
                                    echo "Tests in ${stageName} failed: ${e.getMessage()}"
                                    currentBuild.result = 'UNSTABLE'
                                    throw e // Re-throw to fail the parallel branch
                                } finally {
                                    echo "Stopping and removing ${containerName}..."
                                    sh "docker stop ${containerName} || true"
                                    sh "docker rm ${containerName} || true"

                                    if (fileExists("${agentReportDir}/**/*.xml")) {
                                        junit "${agentReportDir}/**/*.xml"
                                    } else {
                                        echo "Warning: No JUnit XML reports found in ${agentReportDir}"
                                    }
                                }
                            }
                        }
                    }
                    parallel parallelStages
                }
            }
        }
    }

    post {
        always {
            echo "All parallel tests attempted."
            cleanWs()
        }
    }
}