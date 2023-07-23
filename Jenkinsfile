pipeline {
    agent any
    tools {
        jfrog 'jfrog-cli'
    }
 
    stages {
        stage('Build') {
            steps {
                // Trivy scan before git checkout
                sh '/opt/homebrew/bin/trivy repo https://github.com/vdhar71/petclinic.git --scanners vuln,secret,config,license --dependency-tree'
                
                // Get some code from a GitHub repository
                checkout scmGit(branches: [
                    [name: '*/main']
                    ], 
                    extensions: [cleanBeforeCheckout(deleteUntrackedNestedRepositories: true)], 
                    userRemoteConfigs: [
                        [url: 'https://github.com/vdhar71/petclinic']
                        ])
                        
                // Exec Maven commands
                jf '-v'
                jf 'c show'
                jf 'mvn-config --repo-resolve-releases libs-release --repo-resolve-snapshots libs-snapshots --repo-deploy-releases libs-release-local --repo-deploy-snapshots libs-snapshot-local'
                // Check whether dependencies are pulled from JFrog Artifactory
                sh './mvnw -s settings.xml dependency:list'
                // Build petclinic app
                sh './mvnw -s settings.xml package'
                
                // Trivy scan after app is built
                sh '/opt/homebrew/bin/trivy fs . --scanners vuln,secret,config,license --dependency-tree'
                
                // Publish the build info.
                jf 'rt bp'

                
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success {
                    archiveArtifacts 'target/*.jar'
                    // Build the Docker image from the resulting jar
                    sh '/usr/local/bin/docker build -t vdhar/petclinic:1.0 .'
                    
                    // Trivy scan on the final artifact: Docker image
                    sh '/opt/homebrew/bin/trivy image vdhar/petclinic:1.0 --scanners vuln,secret,config,license --dependency-tree'
                    
                    sh '/usr/local/bin/docker save -o petclinic.tar vdhar/petclinic:1.0'
                    jf 'rt u petclinic.tar repo-local/'
                }
            }
        }
    }
}
