# CI/CD pipeline configuration with Tekton and ArgoCD on Kubernetes

Kubernetes 기반의 CI/CD 환경 구축을 위해서 Tekton과 ArgoCD를 사용하여 환경을 구성해보자.

- Tekton installation
- Tekton dashboard installation

## Prerequisites

- Kubernetes cluster version 1.15 or higher and Tekton Pipelines v0.11.0 or higher
- Enable Role-Based Access Control in the cluster
- Grant current user the cluster-admin role

## Tekton dashboard installation

Tekton의 dashboard도 tekton 설치와 동일하게 kubectl로 설치 가능하다. 아래는 Tekton dashboard 설치하는 방법이다.

```console
$ kubectl apply --filename https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml
customresourcedefinition.apiextensions.k8s.io/extensions.dashboard.tekton.dev created
serviceaccount/tekton-dashboard created
role.rbac.authorization.k8s.io/tekton-dashboard-info created
clusterrole.rbac.authorization.k8s.io/tekton-dashboard-backend created
clusterrole.rbac.authorization.k8s.io/tekton-dashboard-tenant created
rolebinding.rbac.authorization.k8s.io/tekton-dashboard-info created
clusterrolebinding.rbac.authorization.k8s.io/tekton-dashboard-backend created
configmap/dashboard-info created
service/tekton-dashboard created
deployment.apps/tekton-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/tekton-dashboard-tenant created
```

Tekton dashboard 설치가 완료되면 정상적으로 pod가 실행되는지 확인하도록 하자. 아래와 같이 tekton-dashboard가 정상적으로 실행되는것을 확인 할 수 있다.

```console
$ kubectl get -n tekton-pipelines pods
NAME                                          READY   STATUS    RESTARTS   AGE
tekton-dashboard-5d44ff59bd-x2lrv             1/1     Running   0          33s
tekton-pipelines-controller-956886f78-pfdzp   1/1     Running   0          125m
tekton-pipelines-webhook-6c6446886c-wx4f6     1/1     Running   0          125m
```

## Configuration Tekton dashboard ingress

dashboard 설치 후 외부 접속을 위한 ingress 설정을 진행하자. 여기에서는 HTTPS를 기본 설정으로 진행됨으로 ingress에서 사용하는 Self Signed Certificate를 먼저 준비하도록 하자. 준비된 인증서로 secret을 생성한다.

```console
$ kubectl create -n tekton-pipelines secret generic tls-secret \
  --from-file=tls.crt=./tekton.homelab.local.crt \
  --from-file=tls.key=./tekton.homelab.local.key
secret/tls-secret created
```

secret 생성이 완료되면 바로 Tekton dashboard를 위한 ingress를 설정하도록 한다.

```console
$ export TEKTON_DASHBOARD_HOSTNAME=tekton.homelab.local
$ kubectl apply -n tekton-pipelines -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tekton-dashboard
  namespace: tekton-pipelines
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: $TEKTON_DASHBOARD_HOSTNAME
    http:
      paths:
      - pathType: ImplementationSpecific
        backend:
          service:
            name: tekton-dashboard
            port:
              number: 9097
  tls:
    - hosts:
        - $TEKTON_DASHBOARD_HOSTNAME
      secretName: tls-secret
EOF
ingress.networking.k8s.io/tekton-dashboard created
```

Tekton dashboard ingress 설정 후 브라우저로 접속하면 정상적으로 접속되는것을 확인 가능하다.

![tekton dashboard](/assets/image/cicd/2022-02-05-tekton_dashboard.png)

## Conclusion

CI/CD Pipeline 구성의 두번째 단계로 tekton daskboard를 구성해 보았다. 다음에는 Tekton의 dashboard 권한 관리를 위해 RBAC 적용 할 것이다.

## References

- [Tekton installation](https://tekton.dev/docs/getting-started/)
