// Jenkinsfile CI with Docker push artifact registry v0.0.0

// Import 구문 위치
import groovy.json.JsonOutput

pipeline {
    agent any
    environment {
        IMAGE_TAG = (sh(script: 'git log -1 --pretty=%s | grep -o "v[0-9]\\+\\.[0-9]\\+\\.[0-9]\\+"', returnStdout: true) ?: 'unknown').trim()
        PROJECT_ID= "GCP_PROJECT_ID"
        ARTIFACT_REGISTRY_LOCATION = "asia-northeast3-docker.pkg.dev"
        ARTIFACT_REGISTRY_REPOSITORY = "GCP_ARTIFACT_REGISTRY"
        DOCKER_IMAGE = "ARTIFACT_REGISTRY_DOCKER_IMAGE"
        ARTIFACT_REGISTRY_IMAGE = "${ARTIFACT_REGISTRY_LOCATION}/${PROJECT_ID}/${ARTIFACT_REGISTRY_REPOSITORY}/${DOCKER_IMAGE}"
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // Docker 빌드
                    docker.build(env.DOCKER_IMAGE + ':' + env.IMAGE_TAG, './ci-app')
                }
            }
        }
        
        stage('Test for Application') {
            steps {
                // CI 테스트 실행
                script {
                    sh 'echo "############### CI Test App ###############"'
                    sh "docker run --rm -d --name ${DOCKER_IMAGE}_${IMAGE_TAG} ${DOCKER_IMAGE}:${IMAGE_TAG} python3 ./app.py" // 컨테이너를 백그라운드에서 실행하고 이름을 지정
                    try {
                        // 여기서 추가적인 테스트 작업을 수행
                        // 예: curl이나 다른 HTTP 요청을 사용하여 테스트를 수행

                    } catch (Exception e) {
                        // 테스트 실패나 예외가 발생했을 경우 처리
                        // 여기에 예외 처리 로직 추가
                    } finally {
                        // 컨테이너를 중지하고 삭제
                        sh "docker stop ${DOCKER_IMAGE}_${IMAGE_TAG}"
                        sh 'echo "############### CI Test DONE###############"'
                    }
                }
            }
        }

        stage('Send Approval Request') {
            steps {
                script {
                    sendTeamsApprovalAndProcess()
                }
            }
        }

        stage('Configure Docker-credential-gcr') {
            steps {
                script {
                    sh "############### Configure docker-credential-gcr ##############"
                    sh 'docker-credential-gcr configure-docker --registries=asia-northeast3-docker.pkg.dev'
                    sh 'echo "https://asia-northeast3-docker.pkg.dev" | docker-credential-gcr get'
                    sh "############### Docker-credential-gcr Done ###################"
                }
            }
        }
        
        stage('Push to Artifact Registry') {
            steps {
                script {
                    def sourceImage = "${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"
                    def targetImage = "${env.ARTIFACT_REGISTRY_IMAGE}:${env.IMAGE_TAG}"
                    
                    // Docker 이미지를 태깅
                    sh(script: "docker tag ${sourceImage} ${targetImage}", returnStatus: true)
                    
                    // 태그된 이미지를 GCP Artifact Registry에 푸시
                    sh(script: "docker push ${targetImage}", returnStatus: true)
                }
            }
        }
        stage('Modify Deployment New Image Tag') {
            steps{
                script{
                    def targetImage = "${env.ARTIFACT_REGISTRY_IMAGE}:${env.IMAGE_TAG}"
                    def cdRepository = "k8s-dev-app-cd"
                    def repoDir= "./${cdRepository}"
                    def repoExists = fileExists(repoDir)
                    
                    if (repoExists) {
                        echo "Repository already exists in Jenkins container. Skipping clone."
                        sh "rm -rf ${cdRepository}"
                        sh "git clone ssh://git@BITBUCKET_OR_GITHUB_URL:PORT/g-cd/${cdRepository}.git"
                    } else {
                        // 저장소가 없으면 git clone
                        sh "git clone ssh://git@BITBUCKET_OR_GITHUB_URL:PORT/g-cd/${cdRepository}.git"
                    }
                    // Image Tag를 새로운 태그로 변경
                    def currentImageTag = sh(script: "cat ./${cdRepository}/deployment.yaml | grep 'image:' | awk '{print \$2}'", returnStdout: true).trim()
                    sh "sed -i 's|${currentImageTag}|${targetImage}|g\' ./${cdRepository}/deployment.yaml"
                    // 변경된 Deployment YAML 파일 확인
                    sh "cat ./${cdRepository}/deployment.yaml"
                    // 변경된 파일을 git에 추가, 커밋
                    sh "cd ./${cdRepository} && git add deployment.yaml && git commit -m 'Update image tag to ${targetImage}' && git push origin main"
                    // 새로운 태그를 만들고 push ( GIT TAG PUSH 사용시 필요함.)
                    // sh "cd ./${cdRepository} && git tag -a ${env.IMAGE_TAG} -m 'Release ${env.IMAGE_TAG}' && git push origin ${env.IMAGE_TAG}"
                }
            }
        }
        stage('All Process Done Notification') {
            steps{
                script{
                    def sourceImage = "${env.DOCKER_IMAGE}:${env.IMAGE_TAG}"
                    def targetImage = "${env.ARTIFACT_REGISTRY_IMAGE}:${env.IMAGE_TAG}"
 
                    // Container 내부에 Artifact Registry로 배포된 이미지 삭제
                    sh(script: "docker rmi ${targetImage}", returnStatus: true)
                    // 이미지 푸시가 성공 후 타겟이미지를 삭제한 경우 성공 메시지를 보내고, 실패한 경우 실패 메시지를 보냄
                    if (currentBuild.currentResult == 'SUCCESS') {
                        sendTeamsNotification('All Process Done...(CI(Docker Image Create, Artifact Registry Push)/CD(Image Tag Update))', false)
                    } else {
                        sendTeamsNotification('Check the logs. Find out what issue caused the failure', true)
                    }
                }                
            }
        }
    }
}

def sendTeamsApprovalAndProcess() {
    def teamsWebhookUrl = 'TEAMS_WEBHOOK_URL'
    
    def pipelineUrl = env.BUILD_URL
    def pipelineNumber = env.BUILD_NUMBER
    def message = "Please review the Container Image deployment plan and provide your approval."
    def pipelineAuthor = sh(script: 'git log -1 --pretty=format:%an', returnStdout: true).trim()
    def activityImage = "https://www.jenkins.io/images/logos/actor/actor.png"
    def themeColor = '#800080'
    def payload = [
        themeColor: themeColor,
        text: "Approval Notification: ${message}",
        sections: [
            [
                activityTitle: "Approval Notification",
                activitySubtitle: "Pipeline #${pipelineNumber}",
                activityImage: activityImage,
                facts: [
                    [name: "Message", value: message],
                    [name: "Author", value: pipelineAuthor],
                    [name: "Deployment URL", value: pipelineUrl],
                ]
            ]
        ]
    ]
    
    def response = httpRequest(
        acceptType: 'APPLICATION_JSON',
        contentType: 'APPLICATION_JSON',
        customHeaders: [[name: 'Authorization', value: "Bearer $teamsWebhookUrl"]],
        httpMode: 'POST',
        requestBody: groovy.json.JsonOutput.toJson(payload),
        url: teamsWebhookUrl
    )
    
    if (response.status != 200) {
        echo "Failed to send Teams notification: ${response.status}"
        return
    }
    
    // Approval이나 Reject가 선택될 때까지 대기
    def userInput = input(
        message: 'Select Action: Approve or Reject?',
        parameters: [choice(
            name: 'ACTION',
            choices: ['Approve', 'Reject'],
            description: 'Select Approvement Action'
        )]
    )
    
    // 사용자의 선택에 따라 다음 단계를 수행하거나 종료
    if (userInput == 'Approve') {
        echo 'User approved the deployment.'
        // 다음 단계 수행
    } else {
        echo 'User rejected the deployment.'
        // 종료 또는 다른 처리 수행
    }
}

def sendTeamsNotification(message, isFailure) {
    def teamsWebhookUrl = 'TEAMS_WEBHOOK_URL'
    
    def pipelineUrl = env.BUILD_URL
    def pipelineNumber = env.BUILD_NUMBER
    def pipelineBranch = env.BRANCH_NAME
    def dockerImage    = env.DOCKER_IMAGE
    def pipelineTag    = env.IMAGE_TAG
    def pipelineAuthor = sh(script: 'git log -1 --pretty=format:%an', returnStdout: true).trim()
    def activityImage = isFailure ? "https://www.jenkins.io/images/logos/fire/fire.png" : "https://www.jenkins.io/images/logos/plumber/plumber.png"
    def themeColor = isFailure ? 'FF0000' : '00FF00'  // Red OR Green
    def payload = [
        themeColor: themeColor,
        text: "Continuous Integration Status : ${message}",
        sections: [
            [
                activityTitle: "CI Details",
                activitySubtitle: "CI #${pipelineNumber} on ${dockerImage}:${pipelineTag}",
                activityImage: activityImage,
                facts: [
                    [name: "Status", value: message],
                    [name: "Author", value: pipelineAuthor],
                    [name: "Pipeline URL", value: pipelineUrl],
                    [name: "Bitbucket Branch Tag", value: pipelineTag]
                ]
            ]
        ]
    ]
    
    def response = httpRequest(
        acceptType: 'APPLICATION_JSON',
        contentType: 'APPLICATION_JSON',
        customHeaders: [[name: 'Authorization', value: "Bearer $teamsWebhookUrl"]],
        httpMode: 'POST',
        requestBody: groovy.json.JsonOutput.toJson(payload),
        url: teamsWebhookUrl
    )
    
    if (response.status != 200) {
        echo "Failed to send Teams notification: ${response.status}"
    } else {
        echo "Teams notification sent successfully!"
    }
}