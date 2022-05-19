# Wazuh 설치 가이드

## Pre-requisites

- Kubernetes 클러스터 
- Storage & Network 플러그인

## Overview

### Pods

#### Wazuh manager master

Wazuh 클러스터의 마스터 노드 역할을 하는 pod. 마스터는 Wazuh 워커 노드를 조율해 클러스터 전반의 data consistency 를 보장하도록 동작.
Agent 등록 서비스와 Wazuh API 서비스 포함

Details:
- Image: Docker Hub 'wazuh/wazuh-manager:4.3.1'
- Controller: StatefulSet


#### Wazuh manager worker

Wazuh 클러스터의 워커 노드 역할을 하는 pod. Agent 로부터 이벤트 수신.

Details:
- Image: Docker Hub 'wazuh/wazuh-manager:4.3.1'
- Controller: StatefulSet


#### Wazuh indexer

Wazuh indexer 컴포넌트 pod (기존의 Elasticsearch). Wazuh indexer 스택 배포에 사용됨.

Details:
- Image: wazuh/wazuh-indexer:4.3.1
- Controller: StatefulSet


#### Wazuh dashboard

Wazuh dashboard 컴포넌트 pod (기존의 Kibana). Wazuh indexer 의 데이터를 시각화하고 플러그인을 통해 Wazuh app 제공.

Details:
- image: Docker Hub 'wazuh/wazuh-dashboard:4.3.1'
- Controller: Deployment


### Services


#### Indexer stack

- wazuh-indexer:
  - Wazuh indexer 노드의 통신.
- indexer:
  - Wazuh indexer API. Wazuh dashboard 가 alert 읽기/쓰기에 사용.
- dashboard:
  - Wazuh dashboard 서비스. https://wazuh.your-domain.com:443


#### Wazuh

- wazuh:
  - Wazuh API: wazuh-master.your-domain.com:55000
  - Agent 등록 서비스 (authd): wazuh-master.your-domain.com:1515
- wazuh-workers:
  - Reporting 서비스: wazuh-manager.your-domain.com:1514
- wazuh-cluster:
  - Wazuh manager 노드의 통신


## Deploy


### Step 1: Deployment

Repository 클론.

```BASH
$ git clone -b 4.3 --single-branch https://github.com/tmax-cloud/wazuh.git
$ cd wazuh-kubernetes
```

### Step 3.1: Setup SSL certificates

`wazuh/certs/indexer_cluster/generate_certs.sh` 스크립트 사용해서 Wazuh indexer 클러스터를 위한 self-signed 인증서 생성
`wazuh/certs/dashboard_http/generate_certs.sh` 스크립트 사용해서 Wazuh dashboard 를 위한 self-signed 인증서 생성

생성된 인증서는 secretGenerator 를 통해 `kustomization.yml` 파일에 import 됨

    secretGenerator:
    - name: indexer-certs
        files:
        - certs/indexer_cluster/root-ca.pem
        - certs/indexer_cluster/node.pem
        - certs/indexer_cluster/node-key.pem
        - certs/indexer_cluster/dashboard.pem
        - certs/indexer_cluster/dashboard-key.pem
        - certs/indexer_cluster/admin.pem
        - certs/indexer_cluster/admin-key.pem
        - certs/indexer_cluster/filebeat.pem
        - certs/indexer_cluster/filebeat-key.pem
    - name: dashboard-certs
        files:
        - certs/dashboard_http/cert.pem
        - certs/dashboard_http/key.pem

### Step 3.2: Apply all manifests using kustomize

`envs/local-env/`디렉토리에서 quota 등의 설정 수정 후 아래 커맨드를 통해 배포

```BASH
$ kubectl apply -k envs/local-env/
```

### Verifying the deployment

#### Namespace

```BASH
$ kubectl get namespaces | grep wazuh
wazuh         Active    12m
```

#### Services

```BASH
$ kubectl get services -n wazuh
NAME                  TYPE           CLUSTER-IP       EXTERNAL-IP        PORT(S)                          AGE
dashboard             ClusterIP      xxx.yy.zzz.24    <none>             9200/TCP                         12m
indexer               ClusterIP      xxx.yy.zzz.76    <none>             5601/TCP                         11m
wazuh                 LoadBalancer   xxx.yy.zzz.209   internal-a7a8...   1515:32623/TCP,55000:30283/TCP   9m
wazuh-cluster         ClusterIP      None             <none>             1516/TCP                         9m
wazuh-indexer         ClusterIP      None             <none>             9300/TCP                         12m
wazuh-workers         LoadBalancer   xxx.yy.zzz.26    internal-a7f9...   1514:31593/TCP                   9m
```

#### Deployments

```BASH
$ kubectl get deployments -n wazuh
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
wazuh-dashboard  1         1         1            1           11m
```

#### Statefulsets

```BASH
$ kubectl get statefulsets -n wazuh
NAME                   READY   AGE
wazuh-indexer          3/3     15m
wazuh-manager-master   1/1     15m
wazuh-manager-worker   2/2     15m
```

#### Pods

```BASH
$ kubectl get pods -n wazuh
NAME                              READY   STATUS    RESTARTS   AGE
wazuh-dashboard-57d455f894-ffwsk  1/1     Running   0          6m18s
wazuh-indexer-0                   1/1     Running   0          15m
wazuh-indexer-1                   1/1     Running   0          15m
wazuh-indexer-2                   1/1     Running   0          14m
wazuh-manager-master-0            1/1     Running   0          6m10s
wazuh-manager-worker-0            1/1     Running   0          8m18s
wazuh-manager-worker-1            1/1     Running   0          8m38s
```

#### Accessing Wazuh dashboard

```BASH
$ kubectl get services -o wide -n wazuh
NAME                  TYPE           CLUSTER-IP       EXTERNAL-IP                                                                       PORT(S)                          AGE       SELECTOR
dashboard             LoadBalancer   xxx.xx.xxx.xxx   internal-xxx-yyy.us-east-1.elb.amazonaws.com                                      80:31831/TCP,443:30974/TCP       15m       app=wazuh-kibana
```
