# Airflow cluster setup for multi-tenance and kubernetes operator

Airflow는 프로그래밍 방식으로 워크플로우를 개발하여 스케줄링 및 모니터링하는 플랫폼이다. Airflow를 사용하면 DAG(Directed Acyclic Graph) 형태의 워크플로우 개발이 가능하며 DAG를 연결하여 확장 가능한 Pipeline도 개발 가능하다. 이러한 편의성으로 많은 프로젝트에서 Airflow를 사용하고 있으며 관련 자료도 매우 쉽게 찾을수 있어 레거시 코드 마이그레이션도 매우 쉽게 가능하다. 그리고 Public Cloud에서는 Managed Airflow를 제공하고 있어 비용만 지불한다면 운영 부담도 매우 낮다.

Airflow Cluster를 설치하는 방법은 [공식 홈페이지](https://airflow.apache.org/docs/apache-airflow/stable/installation/index.html)를 통해서 확인할 수 있으며 운영 환경을 고려한다면 Production Docker Image, Helm Chart, Managed Airflow Service를 사용하여 Airflow를 설치하는 방법이 가장 좋은 방법이다. 일반적인 단일 프로젝트에서는 Airflow 홈페이지에서 제공하는 시스템 구조로도 모든 요구사항에 대하여 대응이 가능하지만 GPU Cluster와 같은 대규모 컴퓨팅 리소스를 여러개의 프로젝트 멥버가 공유해서 사용해야 하는 환경에서는 위 구성으로는 많은 제약 사항이있다. 한 예로 Data Engineer와 Data Scientist가 동일한 Airflow Cluster를 사용해야 한다면 DAG에 대한 ACL부터 고민해야한다. 이번 글에서는 Managed Airflow를 사용하는 방식이 아닌 **VM과 K8S가 혼재된 환경**에서 Airflow Cluster를 설치해보고 더 나아가 멀티테넌시 지원 가능한 Airflow Cluster 설정을 진행해보고자 한다.


## Airflow architecture

Airflow 1.10 이상을 사용한다면 Airflow Worker의 리소스 확장과 편의성으로 대부분 Kubernetes Executor를 사용할것이다. 이때의 시스템 구조는 Airflow 홈페이지에서 가이드하는 아래의 이미지와 같으며 Airflow Web Server, Scheduler의 DAG 파일을 Worker Pod와 공유하는 구조이다.  
![arch-diag-kubernetes](https://airflow.apache.org/docs/apache-airflow/stable/_images/arch-diag-kubernetes.png)

Airflow에서 DAG 실행을 위해서는 Web Server, Scheduler 그리고 Worker에 DAG 파일이 공유되어야 하며, Kubernetes Pod로 실행되는 Worker Pod에 DAG 파일을 공유하기 위해서는 Pod Image에 DAG를 저장하는 방법, Persistent Volume을 사용하는 방법 마지막으로 Git Sync를 사용하는 방법 등이 있다. Pod Image와 PVC를 사용하는 방법은 시스템 구축면에서 매우 쉽지만 사용자가 DAG 파일을 수정하기 매우 까다롭기 때문에 일반적으로 많이 사용하지 않으며 Git Sync를 사용하는 방법은 구축과 사용에 매우 쉬운 장점이 있지만 DAG 파일 수정이 필요한 사용자 모두에서 Git 권한을 설정해야하는 단점이 존재한다. Git 브랜치로 분리하여 사용하는 방법도 좋은 대안이 될 수 있겠지만 브랜치를 분리하는 방법은 Git을 공유하는 방법이므로 이 또한 멀티테넌시 측면에서 좋은 방법은 아닌것으로 판단된다.

![](/assets/image/cicd/2022-05-19-airflow_cluster_architecture.png)

### Airflow architecture considering multi-tenancy with S3

이번글에서는 이러한 DAG 공유 문제를 비교적 사용하기 쉬운 S3를 사용하여 Web Server, Scheduler 그리고 Kubernetes Executor 간의 DAG 공유를 해결하였고 Load Balance를 사용하여 Web Server와 Scheduler의 이중화도 구성하였다. 전체적인 System Architecture는 아래 이미지와 같다.


## Airflow cluster installation

Airflow의 Web Server와 Scheduler는 VM 환경에 설치할것이며 Airflow Cluster 구축에 필요한 Load Balancer, Metadata DB 그리고 S3는 이미 구축되어 있다는 가정으로 설치 진행해보도록 한다. 설치는 HA 구성을 고려하여 3대로 결정하였고 모든 설정은 3대 모두 동일하게 실행하도록 한다.


### Airflow web server and scheduler H/W resource

Airflow web server와 scheduler 서버의 사양은 아래와 같으며 web server와 scheduler에서는 별도의 Job이 실행되지 않으므로 높은 H/W Resource를 요구하지 않는다.

|Hostname|OS|IP|Role|vCPU|Memory|Disk|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|airflow1|Rocky 8|10.0.0.120|Web Server, Scheduler|3|8 GiB|50 GiB|
|airflow2|Rocky 8|10.0.0.121|Web Server, Scheduler|3|8 GiB|50 GiB|
|airflow3|Rocky 8|10.0.0.122|Web Server, Scheduler|3|8 GiB|50 GiB|


### Kernel parameter setting

Airflow 관련 프로세스에서 CPU와 Memory를 최대한 사용하기 위해 관련 kernel parameter 설정을 진행하자. /etc/security/limits.d/airflow.conf 파일을 생성하고 관련 항목을 설정한다.

```bash
$ cat <<EOF > /etc/security/limits.d/airflow.conf
airflow    soft    fsize    unlimited
airflow    hard    fsize    unlimited
airflow    soft    cpu      unlimited
airflow    hard    cpu      unlimited
airflow    soft    as       unlimited
airflow    hard    as       unlimited
airflow    soft    memlock  unlimited
airflow    hard    memlock  unlimited
airflow    soft    nofile   65534
airflow    hard    nofile   65534
airflow    soft    rss      unlimited
airflow    hard    rss      unlimited
airflow    soft    nproc    65534
airflow    hard    nproc    65534
EOF
```


### Python installation

Airflow는 파이썬으로 개발되었으므로 python3 버전을 설치하도록 한다.

```bash
$ sudo dnf install -y python38 python38-pip python38-devel python38-requests s3fs-fuse
```


### Firewall setting

Airflow에서 사용할 로컬 방화벽 포트를 오픈합니다.

```bash
$ sudo firewall-cmd --permanent --add-port 8080/tcp
$ sudo firewall-cmd --reload
```

### User creation

Airflow에서 사용할 OS 사용자를 생성한다. 사용자 UID와 GID는 각각 50000과 100을 사용하도록 한다. 50000과 100을 사용하는 이유는 airflow container 이미지에서 사용하는 아이디가 50000과 100이다.

```bash
$ sudo useradd --gid 100 --uid 50000 --create-home --home-dir /opt/airflow airflow
```

### Default directory creation

Airflow에서 사용하는 기본 디렉토리를 생성한다. 모두 airflow 홈디렉토리 아래에 생성하도록 한다. 디렉토리 생성은 airflow 계정으로 생성되도록 한다.

```bash
$ mkdir -p /opt/airflow/configs /opt/airflow/dags /opt/airflow/plugins /opt/airflow/logs
```

### DAGs directory mount from S3

Airflow의 DAG 파일들은 Web Server, Scheduler 그리고 Worker에 공유되어야하며 여기에서는 S3를 사용하여 동기화하기로 하였다. s3fuse를 사용하여 DAG 디렉토리를 동기화하도록 하자.

```bash
$ echo "${S3_ACCESS_KEY_ID}:${S3_SECRET_KEY} > /etc/s3fs
$ chmod 400 /etc/s3fs
$ mount -t fuse.s3fs airflow /opt/airflow/dags fuse.s3fs -o url=http://192.168.11.4:9000 -o passwd_file=/etc/s3fs -o uid=50000 -o gid=100 -o _netdev -o use_path_request_style -o allow_other -o mp_umask=0022 -o umask=0133 -o dbglevel=info

```

### Airflow installation

pip을 이용하여 airflow를 설치하도록 한다. 여기서는 OIDC, Kubernetes, S3를 사용할 예정이므로 추가 패키지로 앞의 3가지 패키지를 지정하였다. 설치는 성능에 따라 대략 5분정도 소요되므로 천천히 기다려보도록 하자.

```bash
$ python3 -m pip install --user --quiet --no-cache-dir --no-warn-script-location \
    "apache-airflow[celery,google_auth,kubernetes,postgres,s3]==2.2.5" \
        --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.2.5/constraints-no-providers-3.8.txt"
```

### Airflow config setting

[Reference #1](#references)를 참고하여 Airflow의 홈디렉토리에 airflow.cfg 파일을 복사하고 기본 설정을 진행한다. 여기에서 수정한 내용은 아래와 같다.

|Parameter|Default value|Changed value|Description|
|:-:|:-:|:-:|:-:|
|dags_folder|/opt/airflow/airflow/dags|/opt/airflow/airflow/dags/s3|DAG directory path|
|default_timezone|utc|Asia/Seoul|timezone|
|executor|SequentialExecutor|KubernetesExecutor|executor type|
|sql_alchemy_conn|sqlite:////opt/airflow/airflow/airflow.db|postgresql+psycopg2://\${DB_USER}:\${DB_PASS}@\${DB_IP}:\${DB_PORT}/\${DB_NAME}|Database Connection String|
|load_examples|True|False|Load example dags|
|base_log_folder|/opt/airflow/airflow/logs|/opt/airflow/airflow/logs|log directory|
|remote_logging|False|True|remote logging for sharing log files|
|remote_log_conn_id||airflow_log|s3 connection name|
|remote_base_log_folder||s3://airflow/logs|s3 bucket path|
|encrypt_s3_logs|False|False|encrypt s3 logs|
|endpoint_url|http://localhost:8080|http://192.168.11.31:8080|L4 ip, port or uri|
|base_url|http://localhost:8080|http://192.168.11.31:8080|L4 ip, port or uri
|default_ui_timezone|utc|Asia/Seoul|ui timezone|
|result_backend|db+postgresql://postgres:airflow@postgres/airflow|postgresql+psycopg2://\${DB_USER}:\${DB_PASS}@\${DB_IP}:\${DB_PORT}/\${DB_NAME}|Database Connection String for result|
|pod_template_file||/opt/airflow/configs/pod_template.yaml|kubernetes executor template pod|
|worker_container_repository||docker.io/apache/airflow|container repository|
|worker_container_tag||2.2.5-python3.8|container tag|
|namespace|default|airflow|kubernetes executor namespace|
|in_cluster|True|False||
|config_file||/opt/airflow/.kube/config|kubectl config file|


### Web server config for OIDC

OIDC 인증을 위해 webserver_config.py 파일을 설정한다. webserver_config.py에 대한 자세한 설명은 [Airflow authentication with RBAC and Keycloak](https://awslife.medium.com/airflow-authentication-with-rbac-and-keycloak-2c34d2012059)을 참조하도록 하자. 여기서는 Airflow 2.2.X에 맞도록 webserver_config.py 파일을 수정하였다.

```python
#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
"""Default configuration for the Airflow webserver"""
import os
import logging
import jwt

from flask import redirect, session
from flask_appbuilder import expose

from airflow.www.fab_security.manager import AUTH_OAUTH
from airflow.www.security import AirflowSecurityManager
from flask_appbuilder.security.views import AuthOAuthView

basedir = os.path.abspath(os.path.dirname(__file__))

log = logging.getLogger(__name__)

# Flask-WTF flag for CSRF
WTF_CSRF_ENABLED = True

# ----------------------------------------------------
# AUTHENTICATION CONFIG
# ----------------------------------------------------
# For details on how to set up each of the following authentication, see
# http://flask-appbuilder.readthedocs.io/en/latest/security.html# authentication-methods
# for details.

AUTH_TYPE = AUTH_OAUTH

# Uncomment to setup Full admin role name
AUTH_ROLE_ADMIN = 'Admin'

# Uncomment to setup Public role name, no authentication needed
AUTH_ROLE_PUBLIC = 'Public'

# Will allow user self registration
AUTH_USER_REGISTRATION = True

# The default user self registration role
AUTH_USER_REGISTRATION_ROLE = "Public"

AUTH_ROLES_MAPPING = {
  "airflow_admin": ["Admin"],
  "airflow_op": ["Op"],
  "airflow_user": ["User"],
  "airflow_viewer": ["Viewer"],
  "airflow_public": ["Public"],
}

MY_PROVIDER = 'keycloak'
OAUTH_PROVIDERS = [
  {
   'name': MY_PROVIDER,
   'icon': 'fa-circle-o',
   'token_key': 'access_token', 
   'remote_app': {
     'client_id': '{{ oidc_client_id }}',
     'client_secret': '{{ oidc_client_secret }}',
     'client_kwargs': {
       'scope': 'email profile'
     },
     'api_base_url': '{{ oidc_base_url }}/',
     'request_token_url': None,
     'access_token_url': '{{ oidc_base_url }}/token',
     'authorize_url': '{{ oidc_base_url }}/auth',
    },
  },
]

class CustomAuthRemoteUserView(AuthOAuthView):
  @expose("/logout/")
  def logout(self):
    """Delete access token before logging out."""
    super().logout()
    return redirect("{{ oidc_base_url }}/logout?redirect_uri={{ airflow_url }}")

class CustomSecurityManager(AirflowSecurityManager):
  authoauthview = CustomAuthRemoteUserView

  def oauth_user_info(self, provider, response):
    if provider == MY_PROVIDER:
      token = response["access_token"]
      me = jwt.decode(token, algorithms="RS256", verify=False)
      # {
      #   "resource_access": { "airflow": { "roles": ["airflow_admin"] }}
      # }
      groups = me["resource_access"]["{{ oidc_client_id }}"]["roles"] # unsafe
      # log.info("groups: {0}".format(groups))
      if len(groups) < 1:
        groups = ["airflow_public"]

      dct = {
        "username": me.get("preferred_username"),
        "email": me.get("email"),
        "first_name": me.get("given_name"),
        "last_name": me.get("family_name"),
        "role_keys": groups,
      }
      log.info("user info: {0}".format(dct))
      return dct
    else:
      return {}

SECURITY_MANAGER_CLASS = CustomSecurityManager
APP_THEME = "simplex.css"
```

### initialize airflow database

airflow에서 사용할 database를 먼저 초기화한다.

```bash
$ airflow db reset --yes
```

### add s3 connection information

airflow 로깅을 위해 사용할 s3 접속 정보를 설정한다.

```bash
$ airflow connections add airflow_log \
     --conn-type s3 \
     --conn-host {{ S3_CONN_HOST }} \
     --conn-port {{ S3_CONN_PORT }} \
     --conn-login {{ S3_CONN_LOGIN }} \
     --conn-password {{ S3_CONN_PASS }} \
     --conn-extra '{ "host": "{{ S3_CONN_URL }}", "aws_access_key_id":"{{ S3_CONN_LOGIN }}", "aws_secret_access_key": "{{ S3_CONN_PASS }}" }'
```

## Kubernetes Executor setting

Airflow worker 역할을 하는 kubernetes executor를 설정해보도록 하자. executor에 대한 전체적인 설정은 Web Server와 Scheduler에 한 설정을 Kubernetes 상에서 동작하도록 설정하는 내용이다. kubernetes executor에 대한 내용은 [Reference #2](#references)의 링크에서 확인 가능하다.

### kubeconfig file setting

airflow web service와 scheduler가 kubernetes에 명령을 내릴수 있도록 kubeconfig 파일 설정하자. kubeconfig 파일은 위의 airflow.cfg의 config_file에 기록된 /opt/airflow/.kube/config에 복사하도록 하자. 그리고 KUBECONFIG 환경변수에 kubeconfig 파일 위치를 설정하도록 하자.

```bash
$ cp $KUBE_CONFIG_FILE /opt/airflow/.kube/config
$ export KUBECONFIG=/opt/airflow/.kube/config
```

### secret, serviceaccount and configmap creation

kubernetes executor에서 사용할 설정들을 생성하도록 한다.

```bash
$ kubectl create secret generic homelab-airflow-s3-secret \
    --namespace airflow \
    --dry-run=client \
    --from-literal=aws_access_key_id={{ S3_CONN_LOGIN }} \
    --from-literal=aws_secret_access_key={{ S3_CONN_PASS }} \
    -o yaml | kubectl apply -f -

$ kubectl create secret generic homelab-airflow-fernet-key \
    --namespace airflow \
    --dry-run=client \
    --from-literal=fernet-key={{ FERNET_KEY }} \
    -o yaml | kubectl apply -f -

$ kubectl create secret generic homelab-airflow-metadata \
    --namespace airflow \
    --dry-run=client \
    --from-literal=connection={{ DATABASE_CONNECTION_STRING }} \
    -o yaml | kubectl apply -f -

$ kubectl create serviceaccount homelab-airflow-worker-serviceaccount \
    --namespace airflow \
    --dry-run=client \
    -o yaml | kubectl apply -f -

$ kubectl create configmap homelab-airflow-config \
    --namespace airflow \
    --dry-run=client \
    --from-file=/opt/airflow/airflow.cfg \
    -o yaml | kubectl apply -f -
```

### copy pod_template yaml to airflow home directory

pod_template 파일을 생성 후 airflow 홈 디렉토리로 복사한다.

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: homelab-airflow-worker
spec:
  containers:
    - name: airflow-worker
      image: docker.io/apache/airflow:2.2.5-python3.8
      imagePullPolicy: IfNotPresent
      env:
        - name: AIRFLOW__CORE__EXECUTOR
          value: LocalExecutor
        # Hard Coded Airflow Envs
        - name: AIRFLOW__CORE__FERNET_KEY
          valueFrom:
            secretKeyRef:
              name: homelab-airflow-fernet-key
              key: fernet-key
        - name: AIRFLOW__CORE__SQL_ALCHEMY_CONN
          valueFrom:
            secretKeyRef:
              name: homelab-airflow-metadata
              key: connection
        - name: AIRFLOW_CONN_AIRFLOW_DB
          valueFrom:
            secretKeyRef:
              name: homelab-airflow-metadata
              key: connection
      securityContext:
        privileged: true
        runAsUser: 50000
      volumeMounts:
        - mountPath: /opt/airflow/dags
          name: airflow-dags
          mountPropagation: Bidirectional
          readOnly: true
        - mountPath: /opt/airflow/logs
          name: airflow-logs
        - mountPath: /opt/airflow/airflow.cfg
          name: airflow-config
          readOnly: true
          subPath: airflow.cfg
    - name: s3fs
      image: docker.io/efrecon/s3fs:1.89
      imagePullPolicy: IfNotPresent
      env:
        - name: AWS_S3_URL
          value: {{ S3_CONN_URL }}
        - name: AWS_S3_BUCKET
          value: {{ S3_BUCKET }}
        - name: AWS_S3_REGION
          value: ""
        - name: UID
          value: "50000"
        - name: GID
          value: "100"
        - name: S3FS_ARGS
          value: use_path_request_style,allow_other,mp_umask=0022,umask=0133,dbglevel=info
        - name: AWS_S3_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: homelab-airflow-s3-secret
              key: aws_access_key_id
        - name: AWS_S3_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: homelab-airflow-s3-secret
              key: aws_secret_access_key
      resources: {}
      securityContext:
        privileged: true
        runAsUser: 0
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /opt/s3fs/bucket
          name: airflow-dags
          mountPropagation: Bidirectional
  restartPolicy: Never
  securityContext:
    runAsUser: 50000
    fsGroup: 100
  serviceAccountName: "homelab-airflow-worker-serviceaccount"
  volumes:
    - name: airflow-dags
      emptyDir: {}
    - name: airflow-logs
      emptyDir: {}
    - name: airflow-config
      configMap:
        name: homelab-airflow-config
```

## Start web server and scheduler

web server와 scheduler 설정이 완료되었으면 web server와 scheduler의 데몬을 설정하고 시작하도록 하자.

### start web server daemon

web server 데몬은 airflow github 페이지를 참조하여 생성하였으며 서비스 등록 후 시작하도록 한다.

**web server 데몬**
```ini
[Unit]
Description=Airflow webserver daemon
After=network.target

[Service]
Environment=AIRFLOW_HOME={{ airflow_home }}
Environment=PATH={{ airflow_home }}/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
User=airflow
Group={{ airflow_group }}
Type=simple
WorkingDirectory={{ airflow_home }}
RuntimeDirectory=airflow
LogsDirectory=airflow
ExecStart={{ airflow_home }}/.local/bin/airflow webserver --pid /run/airflow/webserver.pid
Restart=on-failure
RestartSec=5s
PrivateTmp=true

[Install]
WantedBy=multi-user.target

```

web server 서비스 시작
```bash
$ systemctl daemon-reload
$ systemctl start airflow-webserver
```

### start scheduler daemon

scheduler 데몬은 airflow github 페이지를 참조하여 생성하였으며 서비스 등록 후 시작하도록 한다.

**scheduler 데몬**
```ini
[Unit]
Description=Airflow scheduler daemon
After=network.target

[Service]
Environment=AIRFLOW_HOME={{ airflow_home }}
Environment=PATH={{ airflow_home }}/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
User=airflow
Group={{ airflow_group }}
Type=simple
WorkingDirectory={{ airflow_home }}
RuntimeDirectory=airflow
LogsDirectory=airflow
ExecStart={{ airflow_home }}/.local/bin/airflow scheduler --pid /run/airflow/scheduler.pid
Restart=on-failure
RestartSec=5s
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

scheduler 서비스 시작

```bash
$ systemctl daemon-reload
$ systemctl start airflow-scheduler
```

## Conclusion

Airflow Cluster에 DAG 파일을 공유하기 위한 VM과 Kubernetes가 혼재된 환경에서 설치를 해보았다. Airflow 홈페이지에 이미지, Persistent Volume 그리고 Git Sync를 사용하여 공유하는 방법이 있지만 DAG를 좀 더 쉽게 공유하기 위해 S3를 사용하였다. 모든 환경을 Kubernetes에 구성하는 방법이 훨씬 쉽지만 부득이 하게 Kubernetes를 제한적으로 사용할 수 밖에 없다면 위에서 가이드하는 방법도 좋은 방법일것으로 생각한다.

## References

- [Airflow Setting Configuration Options](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-config.html#setting-configuration-options)
- [Airflow Kubernetes Executor](https://airflow.apache.org/docs/apache-airflow/stable/executor/kubernetes.html#kubernetes-executor)
