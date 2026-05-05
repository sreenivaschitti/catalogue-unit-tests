// Jenkinsfile

// Load the shared library configured in Jenkins (named 'jenkins-test-library')
@Library('jenkins-test-library') _

// Define a map with configuration that we will pass to the shared library step
def configMap = [
    project  : 'roboshop',
    component: 'catalogue'
]

// Simple log to show that the Jenkinsfile started
echo 'Triggering the library pipeline'

// env.BRANCH_NAME is provided by Jenkins (in multibranch or similar jobs)
if (env.BRANCH_NAME?.equalsIgnoreCase('main')) {
    // If branch is 'main', just print a message for now
    echo 'Current branch is main, skipping testPipeline for now (checking later)'
} else {
    // For other branches, call the shared library pipeline
    testPipeline(configMap)
}