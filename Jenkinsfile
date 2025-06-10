pipeline {
    agent any // Jenkins agent (must be a Windows machine with Docker Desktop, configured for Windows Containers)

    parameters {
        intParam(name: 'PARALLEL_COUNT', defaultValue: 3, description: 'Number of parallel Windows containers to run tests in')
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
                    checkout scm // Checkout the source code once

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

                    // Define what each parallel instance will do.
                    // For demonstration, let's just run the same script for each parallel instance.
                    // In a real scenario, you'd vary 'TEST_SCRIPT' or pass specific arguments.
                    for (int i = 0; i < PARALLEL_COUNT; i++) {
                        // You could pass unique arguments to each instance if needed
                        // For example: `TEST_SCRIPT + " --test-group=${i}"`
                        testCommands.add(env.TEST_SCRIPT)
                    }

                    testCommands.eachWithIndex { cmd, i ->
                        def stageName = "Windows Test Instance ${i + 1}"
                        def containerName = "windows-test-${env.BUILD_NUMBER}-${i}"
                        def agentReportDir = "${pwd()}/target/test-results/instance-${i}" // Unique dir on agent

                        parallelStages[stageName] = {
                            // Agent directive for the Docker container
                            agent {
                                docker {
                                    image "${DOCKUR_IMAGE}"
                                    // todo fix the -v
                                    args "-e VERSION=${WINDOWS_VERSION} -e DISK_SIZE=32G --name ${containerName} -v \"${pwd()}:${PROJECT_DIR_IN_CONTAINER}\" -v \"${agentReportDir}:${REPORT_DIR_IN_CONTAINER}\""
                                }
                            }
                            steps {
                                script {
                                    // Create a directory for this instance's results on the Jenkins agent
                                    sh "mkdir -p ${agentReportDir}"

                                    echo "Running tests in ${stageName} container '${containerName}'..."
                                    try {
                                        // Unstash the source code into the mounted volume
                                        // The 'unstash' command works within the Docker context, writing to the mounted volume
                                        unstash 'project-source'

                                        sh """
                                            echo "Starting ${DOCKUR_IMAGE}:${WINDOWS_VERSION} container named ${containerName}..."
                                            docker run -d \\
                                                -e VERSION=${WINDOWS_VERSION} \\
                                                -e DISK_SIZE=32G \\
                                                -e RDP_PORT=3389 \\
                                                -p 3389 \\
                                                --name ${containerName} \\
                                                -v "${pwd()}:${PROJECT_DIR_IN_CONTAINER}" \\
                                                -v "${agentReportDir}:${REPORT_DIR_IN_CONTAINER}" \\
                                                ${DOCKUR_IMAGE}
                                        """
                                        sh "sleep 120" // Give Windows VM time to boot and for network to initialize

                                        echo "Executing tests in ${containerName}..."
                                        // todo run in VNC session
                                        sh """
                                            docker exec ${containerName} ${TEST_SCRIPT}
                                        """

                                    } catch (Exception e) {
                                        echo "Tests in ${stageName} failed: ${e.getMessage()}"
                                        currentBuild.result = 'UNSTABLE' // Mark as unstable if tests fail in one instance
                                        throw e // Re-throw to fail the parallel branch
                                    } finally {
                                        echo "Stopping and removing ${containerName}..."
                                        // Ensure the container is stopped and removed
                                        sh "docker stop ${containerName} || true" // '|| true' prevents pipeline failure if stop fails
                                        sh "docker rm ${containerName} || true"
                                    }
                                }
                            }
                            post {
                                always {
                                    junit "${agentReportDir}/**/*.xml"
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