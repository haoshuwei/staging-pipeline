pipeline {
    
    agent {
        label 'slave-pipeline'
    }
    
    environment {
        IMAGE_REPO = "registry.cn-hangzhou.aliyuncs.com/haoshuwei/${gitlabTargetRepoName}"  // change it to be your docker registry
        IMAGE_TAG = 'UNKNOWN'
        KUBE_NAMESPACE = "staging"
        dingTalkToken = <your dingTalk token>  // change it to be your dingTalk token
    }
    
    options {
        gitLabConnection('gitlab') // change it to be your gitlabConnection name
    }
    stages {
        stage('Fetch Git Repo') {
            when {
                expression {
                    "${gitlabTargetBranch}" == 'latest'
                }
            }
            
            steps {
		// change it to be your credentialsId and url
		git branch: '${gitlabTargetBranch}', changelog: false, credentialsId: 'gitlab', poll: false, url: 'http://xxx.xxx.xxx/application/application-demo.git'
                
                script {
                    GIT_COMMIT_ID = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
		    IMAGE_TAG = "${GIT_COMMIT_ID}-pre"
		    KUBE_NAMESPACE_PREVIEW = "preview-${gitlabSourceRepoName}-${env.gitlabMergeRequestIid}"
                    INGRESS_HOST = "staging-${gitlabSourceRepoName}"
                }
            }
        }
        
        stage('Maven Build') {
            when {
                expression {
                    "${gitlabTargetBranch}" == 'latest'
                }
            }
        
            steps {
                container("maven") {
                    sh("mvn package -B -DskipTests")
                }
            }
        }
        
        stage('Maven Test') {
            when {
                expression {
                    "${gitlabTargetBranch}" == 'latest'
                }
            }
        
            steps {
                container("maven") {
                    sh("mvn test")
                }
            }
        }
        
        stage('Docker Build And Publish') {
            when {
                expression {
                    "${gitlabTargetBranch}" == 'latest'
                }
            }
        
            steps {
                container("kaniko") {
                    sh("kaniko -f `pwd`/Dockerfile -c `pwd` --destination=${IMAGE_REPO}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Kubectl Deploy') {
            when {
                expression {
                    "${gitlabTargetBranch}" == 'latest'
                }
            }
        
            steps {
                container("kubectl") {
		    // change it to be your yaml files path
                    sh("kubectl get ns ${env.KUBE_NAMESPACE} || kubectl create ns ${env.KUBE_NAMESPACE}")
                    sh("sed -i.bak 's#IMAGE_REPO#${IMAGE_REPO}#' ./*.yaml")
                    sh("sed -i.bak 's#IMAGE_TAG#${IMAGE_TAG}#' ./*.yaml")
                    sh("sed -i.bak 's#INGRESS_HOST#${INGRESS_HOST}#' ./*.yaml")
                    sh("kubectl --namespace=${KUBE_NAMESPACE} apply -f ./")
                    script {
                        SERVICE_DOMAIN = sh(returnStdout: true, script: 'kubectl -n ${KUBE_NAMESPACE} get ing application-demo -o jsonpath=\'{.spec.rules[0].host}\'').trim()
                        SERVICE_PATH = sh(returnStdout: true, script: 'kubectl -n ${KUBE_NAMESPACE} get ing application-demo -o jsonpath=\'{.spec.rules[0].http.paths[0].path}\'').trim()
                    }
                }
            }
        }
        
        stage('Clean Preview Env') {
            when {
                expression {
                    "${gitlabTargetBranch}" == 'latest'
                }
            }
        
            steps {
		        container("kubectl") {
		            sh("kubectl --namespace=${KUBE_NAMESPACE_PREVIEW} delete -f ./")
		            sh("kubectl delete ns ${KUBE_NAMESPACE_PREVIEW}")
		        }
            }
        }
    }
    
    post {
        success {
            script {
                if (env.gitlabTargetBranch == 'latest') {
                    sh("curl -X POST 'https://oapi.dingtalk.com/robot/send?access_token=\'${dingTalkToken}\'' \
                    -H 'cache-control: no-cache' \
                    -H 'content-type: application/json' \
                    -d '{\"msgtype\": \"markdown\", \"markdown\": {\"title\": \"Staging Build and Deploy\",\"text\":\"## Build success!  \n ### Environment Type: Staging \n  ### Jenkins Build: [\'$BUILD_DISPLAY_NAME\'](\'${BUILD_URL}\') \n ### App Name: [\'$gitlabSourceRepoName\'](http://\'$SERVICE_DOMAIN\'\'$SERVICE_PATH\') \n ### Commit By: \'$gitlabUserName\'\"}}'")
                }
            }
        }
        failure {
            script {
                if (env.gitlabMergeRequestIid != "" && env.gitlabTargetBranch == 'latest') {
                    sh("curl -X POST 'https://oapi.dingtalk.com/robot/send?access_token=\'${dingTalkToken}\'' \
                    -H 'cache-control: no-cache' \
                    -H 'content-type: application/json' \
                    -d '{\"msgtype\": \"markdown\", \"markdown\": {\"title\": \"Staging Build and Deploy\",\"text\":\"## Build failed!  \n ### Environment Type: Staging \n  ### Jenkins Build: [\'$BUILD_DISPLAY_NAME\'](\'${BUILD_URL}\') \n ### App Name: \'$gitlabSourceRepoName\' \n ### Commit By: \'$gitlabUserName\'\"}}'")
                }
            }
        }
    }
}

