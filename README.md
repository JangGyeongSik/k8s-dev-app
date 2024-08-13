# CI/CD Demo App

CI/CD Demo App 배포를 위한 Repository입니다. 


# Requirement Service 

* Docker
* GCP(docker-credentials-gcr, Artifact Registry, Service Account)
* Jenkins(Jenkinsfile)
* Teams(Approval)
---

# CI/CD Pipeline Architecture 

![CI/CD with Jenkins and ArgoCD](./Jenkins_ArgoCD(Flow_Chart).png)
---

# CI/CD Flow
1. ci-app에 변경사항이 있을경우, Commit후 Tag Push를 진행합니다.
    ```sh
    git add && git commit -m "[UPDATE] vx.x.x" 
    ```
2. 이후 Bitbucket Webhook Trigger에 의해 CI는 자동으로 진행됩니다.

3. Docker Build 및 Application Test를 진행합니다.

4. Test가 완료되면 MS Teams를 통해 Approval Message를 받게됩니다.

5. Approval을 선택하게되면 GCP Artifact Registry로 Docker Image를 Push합니다. 

6. GitOps Repo에 신규로 배포된 Docker Image의 Tag로 Deployment를 변경해줍니다.
    ```yaml
    containers:
    - name: CONTAINER_NAME
    image: ARTIFACT_REGISTRY_APP_PATH:GIT_IMAGE_TAG_NEW
    ports:
    - containerPort: CONTAINER_PORT
    ```
7. ArgoCD에서 GitOps를 통해 App에 OutOfSync Message를 확인합니다.

8. App 배포 이전에 KSA(Kubernetes Service Account)가 Workload Identity를 통해서 Artifact Registry에 권한을 GSA(Google Service Account)에 위임하여 Docker Image를 Pull 해옵니다. 

9. DEV Cluster의 경우, Automate Deploy 설정을 통해 App이 GKE의 특정 Namespace에 배포됩니다. 

10. PROD Cluster의 경우, Manual Deploy를 통해 Application 담당자가 수동 배포를 진행하여 GKE의 특정 Namespace에 배포합니다. 

11. 모든 작업이 끝나게되면, ArgoCD Notification Controller가 Teams로 ArgoCD의 App Sync 상태를 전달하여 App 배포 상태를 확인합니다. 
---