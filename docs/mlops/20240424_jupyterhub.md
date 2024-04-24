---
title: JupyterHub 설치
layout: default
nav_exclude: true
parent: MLOps
---

# JupyterHub 설치

## Install

### Helm repo 설치

```sh
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update
```

### values 수정

`config.yaml` 파일을 생성하고 커스텀 values를 정의한다.

```sh
touch config.yaml
```

참고: [values.yaml](https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/main/jupyterhub/values.yaml)

#### 이미지 선정

[Jupyter 이미지 선택](https://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html)의 가이드를 따라 상황에 맞게 이미지를 선정할 수 있다.

많이 쓰이는 아래의 이미지 중

- jupyter/minimal-notebook: 최소 스펙 노트북
- jupyter/datascience-notebook: 데이터 분석을 위한 노트북
- jupyter/tensorflow-notebook: tensorflow가 포함된 노트북
- jupyter/pyspark-notebook: pyspark가 포함된 노트북

jupyter/minimal-notebook을 default로 설정한다.

```yaml
singleuser:
  image:
    name: jupyter/minimal-notebook
    tag: hub-4.0.2
```

유저들에게 다양한 선택지를 제공하기 위해서 노트북 환경을 선택할 수 있게 옵션을 추가한다.

```yaml
singleuser:
  image:
    name: jupyter/minimal-notebook
    tag: hub-4.0.2
  profileList:
    - display_name: "minimal"
      default: true
    - display_name: "data science"
      kubespawner_override:
        image: jupyter/datascience-notebook:hub-4.0.2
    - display_name: "tensorflow"
      kubespawner_override:
        image: jupyter/tensorflow-notebook:hub-4.0.2
    - display_name: "pyspark"
      kubespawner_override:
        image: jupyter/pyspark-notebook:hub-4.0.2
```

### Auth

Google, GitHub 혹은 LDAP 인증을 추가하여 유저 관리를 할 수 있다.

```yaml
hub:
  config:
    GoogleOAuthenticator:
      client_id: <GOOGLE_CLIENT_ID>
      client_secret: <GOOGLE_CLIENT_SECRET>
      oauth_callback_url: https://<DOMAIN>/hub/oauth_callback
      hosted_domain:
        - <HOSTED_DOMAIN>
      admin_users:
        - <ADMIN_USER>
      login_service: <SERVICE_NAME>
    JupyterHub:
      authenticator_class: google
```

- `GOOGLE_CLIENT_ID`: GCP > APIs & Services > Credentials 에서 발급한 client id
- `GOOGLE_CLIENT_SECRET`: GCP > APIs & Services > Credentials 에서 발급한 client secret
- `DOMAIN`: JupyterHub이 배포될 도메인 url (e.g. jupyterhub.80rian.com)
- `HOSTED_DOMAIN`: Google 계정의 도메인 (e.g. 80rian.com)
- `ADMIN_USER`: 어드민으로 지정될 유저의 Google 계정 아이디 (e.g. adrian.jeong)
- `SERVICE_NAME`: 유저 서비스 이름 (e.g. 80rian )

### Service Account

service account를 annotate해서 JupyterHub pod에 권한을 주입한다. IAM에서 `jupyterhub-role`이라는 role을 만들어 사용한다.

```yaml
hub:
  serviceAccount:
    create: true
    name: hub
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT>:role/jupyterhub-role
```

### Environment Variables

전사에서 공통적으로 사용하는 변수들을 지정해준다.

```yaml
singleuser:
  extraEnv:
    HOST: "80rian.com"
```

정의된 변수는 `os.environ["HOST"]`로 호출할 수 있다.

### Resources & Storage

pod의 resource와 storage도 설정할 수 있다.

```yaml
singleuser:
  cpu:
    gurantee: 0.5
    limit: 1
  memory:
    guarantee: 1G
    limit: 5G
  storage:
    capacity: 10Gi
```

### Ingress

alb 배포를 위한 ingress 설정을 한다.

```yaml
ingress:
  enabled: true
  annotations:
    {
      alb.ingress.kubernetes.io/certificate-arn: <CERTIFICATE_ARN>,
      alb.ingress.kubernetes.io/inbound-cidrs: 0.0.0.0/0,
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]',
      alb.ingress.kubernetes.io/scheme: internal,
      alb.ingress.kubernetes.io/target-type: ip,
    }
  ingressClassName: alb
  hosts:
    - <HOST>
  pathSuffix:
  pathType: Prefix
```

JupyterHub과 같은 데이터 인프라는 외부에 노출되지 않아야 하기 때문에 internal로 배포한다.

## Deploy

```sh
helm upgrade --install jupyterhub jupyterhub/jupyterhub \
  --cleanup-on-fail \
  --create-namespace \
  -n jupyterhub \
  -f ./config.yaml
```

## References

- [Zero to Kubernetes JupyterHub Docs](https://z2jh.jupyter.org/en/stable/index.html)
- [JupyterHub GitHub](https://github.com/jupyterhub/zero-to-jupyterhub-k8s)
