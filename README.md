# Wazuh 설치 가이드

## 구성 요소 및 버전
- wazuh-indexer
	- image: [wazuh/wazuh-indexer:4.3.1](https://hub.docker.com/r/wazuh/wazuh-indexer)
- wazuh-dashboard
	- image: [wazuh/wazuh-dashboard:4.3.1](https://hub.docker.com/r/wazuh/wazuh-dashboard)
- wazuh-managers
	- image: [wazuh/wazuh-manager:4.3.1](https://hub.docker.com/r/wazuh/wazuh-manager)

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
  - Wazuh manager 노드의 통신.


## Deploy


### Step 1: Deployment

Repository 클론.

```BASH
$ git clone -b 4.3 --single-branch https://github.com/tmax-cloud/wazuh.git
$ cd wazuh-kubernetes
```

### Step 3.1: Setup SSL certificates

You can generate self-signed certificates for the ODFE cluster using the script at `wazuh/certs/odfe_cluster/generate_certs.sh` or provide your own.

Since Kibana has HTTPS enabled it will require its own certificates, these may be generated with: `openssl req -x509 -batch -nodes -days 365 -newkey rsa:2048 -keyout key.pem -out cert.pem`, there is an utility script at `wazuh/certs/kibana_http/generate_certs.sh` to help with this.

The required certificates are imported via secretGenerator on the `kustomization.yml` file:

    secretGenerator:
    - name: odfe-ssl-certs
        files:
        - certs/odfe_cluster/root-ca.pem
        - certs/odfe_cluster/node.pem
        - certs/odfe_cluster/node-key.pem
        - certs/odfe_cluster/kibana.pem
        - certs/odfe_cluster/kibana-key.pem
        - certs/odfe_cluster/admin.pem
        - certs/odfe_cluster/admin-key.pem
        - certs/odfe_cluster/filebeat.pem
        - certs/odfe_cluster/filebeat-key.pem
    - name: kibana-certs
        files:
        - certs/kibana_http/cert.pem
        - certs/kibana_http/key.pem

### Step 3.2: Apply all manifests using kustomize

We are using the overlay feature of kustomize to create two variants: `eks` and `local-env`, in this guide we're using `eks`. (For a deployment on a local environment check the guide on [local-environment.md](local-environment.md))

You can adjust resources for the cluster on `envs/eks/`, you can tune cpu, memory as well as storage for persistent volumes of each of the cluster objects.


By using the kustomization file on the `eks` variant we can now deploy the whole cluster with a single command:

```BASH
$ kubectl apply -k envs/eks/
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
elasticsearch         ClusterIP      xxx.yy.zzz.24    <none>             9200/TCP                         12m
kibana                ClusterIP      xxx.yy.zzz.76    <none>             5601/TCP                         11m
wazuh                 LoadBalancer   xxx.yy.zzz.209   internal-a7a8...   1515:32623/TCP,55000:30283/TCP   9m
wazuh-cluster         ClusterIP      None             <none>             1516/TCP                         9m
wazuh-elasticsearch   ClusterIP      None             <none>             9300/TCP                         12m
wazuh-workers         LoadBalancer   xxx.yy.zzz.26    internal-a7f9...   1514:31593/TCP                   9m
```

#### Deployments

```BASH
$ kubectl get deployments -n wazuh
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
wazuh-kibana     1         1         1            1           11m
```

#### Statefulsets

```BASH
$ kubectl get statefulsets -n wazuh
NAME                   READY   AGE
wazuh-elasticsearch    3/3     15m
wazuh-manager-master   1/1     15m
wazuh-manager-worker   2/2     15m
```

#### Pods

```BASH
$ kubectl get pods -n wazuh
NAME                            READY   STATUS    RESTARTS   AGE
wazuh-elasticsearch-0           1/1     Running   0          15m
wazuh-elasticsearch-1           1/1     Running   0          15m
wazuh-elasticsearch-2           1/1     Running   0          14m
wazuh-kibana-7c9657f5c5-z95pt   1/1     Running   0          6m18s
wazuh-manager-master-0          1/1     Running   0          6m10s
wazuh-manager-worker-0          1/1     Running   0          8m18s
wazuh-manager-worker-1          1/1     Running   0          8m38s
```

#### Accessing Kibana

In case you created domain names for the services, you should be able to access Kibana using the proposed domain name: https://wazuh.your-domain.com.

Also, you can access using the External-IP (from the VPC): https://internal-xxx-yyy.us-east-1.elb.amazonaws.com:443

```BASH
$ kubectl get services -o wide -n wazuh
NAME                  TYPE           CLUSTER-IP       EXTERNAL-IP                                                                       PORT(S)                          AGE       SELECTOR
kibana                LoadBalancer   xxx.xx.xxx.xxx   internal-xxx-yyy.us-east-1.elb.amazonaws.com                                      80:31831/TCP,443:30974/TCP       15m       app=wazuh-kibana
```
