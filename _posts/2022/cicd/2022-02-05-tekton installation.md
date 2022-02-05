# CI/CD pipeline configuration with tekton and argocd on Kubernetes

Kubernetes 기반의 CI/CD 환경 구축을 위해서 tekton과 argocd를 사용하여 환경을 구성해보자.

- 1. Tekton installation
- 2. Tekton dashboard installation

## Prerequisites

- Kubernetes cluster version 1.15 or higher and Tekton Pipelines v0.11.0 or higher
- Enable Role-Based Access Control in the cluster
- Grant current user the cluster-admin role

## Tekton Installation

tekton의 core component 설치를 위해서는 Tekton Pipelines 설치가 필요하다. 설치는 아래 명령어로 설치 가능하다.

```console
$ kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
namespace/tekton-pipelines created
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/tekton-pipelines created
clusterrole.rbac.authorization.k8s.io/tekton-pipelines-controller-cluster-access created
clusterrole.rbac.authorization.k8s.io/tekton-pipelines-controller-tenant-access created
clusterrole.rbac.authorization.k8s.io/tekton-pipelines-webhook-cluster-access created
role.rbac.authorization.k8s.io/tekton-pipelines-controller created
role.rbac.authorization.k8s.io/tekton-pipelines-webhook created
role.rbac.authorization.k8s.io/tekton-pipelines-leader-election created
role.rbac.authorization.k8s.io/tekton-pipelines-info created
serviceaccount/tekton-pipelines-controller created
serviceaccount/tekton-pipelines-webhook created
clusterrolebinding.rbac.authorization.k8s.io/tekton-pipelines-controller-cluster-access created
clusterrolebinding.rbac.authorization.k8s.io/tekton-pipelines-controller-tenant-access created
clusterrolebinding.rbac.authorization.k8s.io/tekton-pipelines-webhook-cluster-access created
rolebinding.rbac.authorization.k8s.io/tekton-pipelines-controller created
rolebinding.rbac.authorization.k8s.io/tekton-pipelines-webhook created
rolebinding.rbac.authorization.k8s.io/tekton-pipelines-controller-leaderelection created
rolebinding.rbac.authorization.k8s.io/tekton-pipelines-webhook-leaderelection created
rolebinding.rbac.authorization.k8s.io/tekton-pipelines-info created
customresourcedefinition.apiextensions.k8s.io/clustertasks.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/conditions.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/pipelines.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/pipelineruns.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/pipelineresources.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/runs.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/tasks.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/taskruns.tekton.dev created
secret/webhook-certs created
validatingwebhookconfiguration.admissionregistration.k8s.io/validation.webhook.pipeline.tekton.dev created
mutatingwebhookconfiguration.admissionregistration.k8s.io/webhook.pipeline.tekton.dev created
validatingwebhookconfiguration.admissionregistration.k8s.io/config.webhook.pipeline.tekton.dev created
clusterrole.rbac.authorization.k8s.io/tekton-aggregate-edit created
clusterrole.rbac.authorization.k8s.io/tekton-aggregate-view created
configmap/config-artifact-bucket created
configmap/config-artifact-pvc created
configmap/config-defaults created
configmap/feature-flags created
configmap/pipelines-info created
configmap/config-leader-election created
configmap/config-logging created
configmap/config-observability created
configmap/config-registry-cert created
deployment.apps/tekton-pipelines-controller created
service/tekton-pipelines-controller created
horizontalpodautoscaler.autoscaling/tekton-pipelines-webhook created
deployment.apps/tekton-pipelines-webhook created
service/tekton-pipelines-webhook created
```

정상적으로 설치되었는지 아래의 명령어로 확인 가능하다.

```console
$ kubectl --namespace tekton-pipelines get pods
NAME                                          READY   STATUS    RESTARTS   AGE
tekton-pipelines-controller-956886f78-pfdzp   1/1     Running   0          48s
tekton-pipelines-webhook-6c6446886c-wx4f6     1/1     Running   0          48s
```

## Persistent volumes

CI/CD workflow 실행을 위해서는 Tekton을 위한 저장 공간을 추가로 구성해주어야 한다. Tekton에서는 필요로 하는 저장 공간을 PV 또는 Object Storage로 설정하여 사용 가능하며 이때 필요로 하는 최소 저장 공간은 5Gi이다. 이 글에서는 PV 보다는 Object Storage를 사용하여 저장 공간을 구성하였다.

```console
$ export S3_CONN='http://IP:PORT/BUCKET'
$ export S3_ACCKEY='awslife'
$ export S3_SECKEY='Super$tr0ng'
$ kubectl create configmap config-artifact-pvc \
                         --from-literal=location=${S3_CONN} \
                         --from-literal=bucket.service.account.secret.name=${S3_ACCKEY} \
                         --from-literal=bucket.service.account.secret.key=${S3_SECKEY} \
                         -o yaml -n tekton-pipelines \
                         --dry-run=client | kubectl replace -f -
$ kubectl --namespace tekton-pipelines get configmaps config-artifact-pvc -o yaml
apiVersion: v1
data:
  bucket.service.account.secret.key: Super$tr0ng
  bucket.service.account.secret.name: awslife
  location: http://IP:PORT/BUCKET
kind: ConfigMap
metadata:
  creationTimestamp: "2022-02-05T06:30:10Z"
  name: config-artifact-pvc
  namespace: tekton-pipelines
  resourceVersion: "41777"
  uid: d87e8465-df86-4721-99e9-0bd2551ae98d
```

추가로 tekton에서 사용하는 service account를 변경하고 싶으면 아래와 같이 configmap을 수정하여 service account 변경이 가능하다. 변경하지 않으면 기본적으로 default 계정을 사용한다.

```console
$ kubectl create configmap config-defaults \
                         --from-literal=default-service-account=awslife \
                         -o yaml -n tekton-pipelines \
                         --dry-run=client  | kubectl replace -f -
```

## Set up the CLI

Tekton을 좀 더 쉽게 사용하기 위해서 Tekton CLI도 추가로 구성해주기로 한다. 개발 환경을 Mac을 사용하므로 brew를 통하여 Tekton CLI를 구성하였다.

```console
$ brew install tektoncd-cli
==> Downloading https://ghcr.io/v2/homebrew/core/tektoncd-cli/manifests/0.22.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/tektoncd-cli/blobs/sha256:2f4b8897725c6144db96b15408600cecd475c57dd401b7aaae6dea55c566e131
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sha256:2f4b8897725c6144db96b15408600cecd475c57dd401b7aaae6dea55c566e131?se=2022-02-05T07%3A55%3A00Z&sig=l3uB%2FXsVv542ImqEJjt6XxVcVgYyWBBVUCwOhFGKJyg%3D&sp=r&spr=https&sr=b&sv=2019-12-12
######################################################################## 100.0%
==> Pouring tektoncd-cli--0.22.0.monterey.bottle.tar.gz
==> Caveats
zsh completions have been installed to:
  /usr/local/share/zsh/site-functions
==> Summary
🍺  /usr/local/Cellar/tektoncd-cli/0.22.0: 8 files, 75.9MB
==> Running `brew cleanup tektoncd-cli`...
```

## Your first CI/CD workflow with Tekton

Tekton 구성이 완료되었으면 간단한 예제를 사용하여 테스트 해보도록 한다.

### Create Task

Tekton workflow에서는 먼저 Task를 생성해야 한다. 아래 명령을 실행하여 hello task를 생성하도록 하자.

```console
$ kubectl apply -f - << EOF
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: hello
      image: ubuntu
      command:
        - echo
      args:
        - "Hello World!"
EOF

task.tekton.dev/hello created
```

### Run task

생성된 Task를 실행하기 위해서는 TaskRun을 생성해주어야 한다. Tekton CLI를 이용하여 TaskRun을 실행하자.

```console
$ tkn task start hello
TaskRun started: hello-run-rspzl

In order to track the TaskRun progress run:
tkn taskrun logs hello-run-rspzl -f -n kube-system
```

TaskRun이 생성 완료되었다면 hello-run의 로그를 확인해보도록 하자.

```console
$ tkn taskrun logs hello-run-rspzl -f -n kube-system
[hello] Hello World!
```

## Conclusion

CI/CD Pipeline 구성의 첫번째 단계로 tekton 구성해 보았고 간단한 예제로 구성 상태를 확인해 보았다. 다음에는 GUI를 통하여 Task 확인이 가능한 dashboard 구성 방법을 알아보도록 하자

## References

- [Tekton installation](https://tekton.dev/docs/getting-started/)
