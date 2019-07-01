pipeline {

    agent {
        label 'slave-pipeline'
    }

    stages {

        stage('Fetch Git Repo') {
            when {
                expression {
                    "${ghprbTargetBranch}" == 'latest'
                }
            }

            steps {
		// change it to be your credentialsId and url
		git branch: '${ghprbTargetBranch}', changelog: false, credentialsId: '', poll: false, url: 'https://github.com/haoshuwei/jenkins-demo.git'
            }
        }

        stage('Maven Build') {
            steps {
                container("maven") {
	            sh("git branch")
                    sh("cat README.md")
                }
            }
        }

    }

}
