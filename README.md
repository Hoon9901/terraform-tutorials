# Terraform Tutorials
Kubernetes(K8S)는 컨테이너화된 애플리케이션에 중점을 둔 오픈 소스 워크로드 스케줄러입니다. Terraform Kubernetes provider를 사용하여 Kubernetes에서 지원하는 리소스와 상호 작용할 수 있습니다.  
  
이 문서에서는 Kubernetes 클러스터에서 NGINX 배포를 예약하고 노출하여 Terraform을 사용하여 Kubernetes와 상호 작용하는 방법을 배우고 Terraform을 사용하여 사용자 지정 리소스(CRD)를 관리합니다.  

# Why Deploy with Terraform
kubectl 또는 유사한 CLI 기반 도구를 사용하여 Kubernetes 리소스를 관리할 수 있지만 Terraform을 사용하면 다음과 같은 이점이 있습니다.  
  
- 통합 워크플로 - 이미 Terraform으로 Kubernetes 클러스터를 프로비저닝하고 있는 경우 동일한 구성 언어를 사용하여 애플리케이션을 클러스터에 배포합니다.  
  
- 전체 수명 주기 관리 - Terraform은 리소스를 생성할 뿐만 아니라 해당 리소스를 식별하기 위해 API를 검사할 필요 없이 추적된 리소스를 업데이트 및 삭제합니다.  
  
- 관계 그래프 - Terraform은 리소스 간의 종속성 관계를 이해합니다. 
- 예를 들어 특정 PVC가 특정 PV의 공간을 요구하는 경우 Terraform은 볼륨 생성에 실패하면 클레임 생성을 시도하지 않습니다.

# Configure the provider
Terraform을 사용하여 Kubernetes 서비스를 예약하려면 먼저 Terraform Kubernetes provider를 구성해야 합니다.  
  
Kubernetes provider를 구성하는 방법에는 여러 가지가 있습니다. 

다음 순서로 권장합니다(가장 많이 권장되는 항목이 먼저, 가장 적게 권장되는 항목이 마지막으로 표시됨).
1.  클라우드별 인증 플러그인 사용(예: eks get-token, az get-token, gcloud config)
2.  [oauth2 token](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/guides/getting-started#provider-setup) 사용
3.  [TLS certificate credentials](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#statically-defined-credentials) 사용
4.  [`config_path`](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#config_path) 와 [`config_context`](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#config_context) 모두 설정하여 `kubeconfig` 파일 사용
5.  [username and password (HTTP Basic Authorization)](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#statically-defined-credentials) 사용

종류 또는 클라우드 provider 탭의 지침에 따라 provider가 특정 Kubernetes 클러스터를 대상으로 지정하도록 구성합니다. 

클라우드 provider 탭은 클라우드별 인증 토큰을 사용하여 Kubernetes provider를 구성합니다.

- kubernetes.tf
```tf
terraform {
  required_providers {
    kubernetes = {
      source = "hashicorp/kubernetes"
    }
  }
}

variable "host" {
  type = string
}

variable "client_certificate" {
  type = string
}

variable "client_key" {
  type = string
}

variable "cluster_ca_certificate" {
  type = string
}

provider "kubernetes" {
  host = var.host

  client_certificate     = base64decode(var.client_certificate)
  client_key             = base64decode(var.client_key)
  cluster_ca_certificate = base64decode(var.cluster_ca_certificate)
}
```
이 provider를 올바르게 구성하려면 변수를 정의해야 합니다.  
  
먼저 종류 클러스터 정보를 봅니다.
```sh 
$ kubectl config view --minify --flatten --context=kind-terraform-learn
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRU...
    server: https://127.0.0.1:32768
  name: kind-terraform-learn
contexts:
- context:
    cluster: kind-terraform-learn
    user: kind-terraform-learn
  name: kind-terraform-learn
current-context: kind-terraform-learn
kind: Config
preferences: {}
users:
- name: kind-terraform-learn
  user:
    client-certificate-data: LS0tLS1CRU...
    client-key-data: LS0tLS1CRU...
```


terraform.tfvars 파일에서 변수를 정의합니다.  

- `host`는 `cluster.cluster.server`에 해당합니다.  
- `client_certificate`는 `users.user.client-certificate`에 해당합니다.  
- `client_key`는 `users.user.client-key`에 해당합니다.  
- `cluster_ca_certificate`는 `clusters.cluster.certificate-authority-data`에 해당합니다.
```tf
host                   = "https://127.0.0.1:32768"
client_certificate     = "LS0tLS1CRUdJTiB..."
client_key             = "LS0tLS1CRUdJTiB..."
cluster_ca_certificate = "LS0tLS1CRUdJTiB..."
```

provider를 구성한 후 `terraform init`를 실행하여 최신 버전을 다운로드하고 Terraform 작업 영역을 초기화합니다.

## 추가적으로 Token 방식이면
```tf
provider "kubernetes" {
  host                   = "https://${data.google_container_cluster.my_cluster.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(data.google_container_cluster.my_cluster.master_auth[0].cluster_ca_certificate)
}
```
- host
- token
- cluster_ca_certificate

# Schedule a deployment
kubernetes.tf 파일에 다음을 추가합니다. 

이 Terraform 구성은 내부적으로 포트 80(HTTP)을 노출하는 Kubernetes 클러스터에 두 개의 Replica이 있는 NGINX deployments를 예약합니다.
```tf
resource "kubernetes_deployment" "nginx" {
  metadata {
    name = "scalable-nginx-example"
    labels = {
      App = "ScalableNginxExample"
    }
  }

  spec {
    replicas = 2
    selector {
      match_labels = {
        App = "ScalableNginxExample"
      }
    }
    template {
      metadata {
        labels = {
          App = "ScalableNginxExample"
        }
      }
      spec {
        container {
          image = "nginx:1.7.8"
          name  = "example"

          port {
            container_port = 80
          }

          resources {
            limits = {
              cpu    = "0.5"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "50Mi"
            }
          }
        }
      }
    }
  }
}

```
Terraform 구성과 Kubernetes 구성 [YAML 파일](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/) 간의 유사성을 알 수 있습니다.


구성을 적용하여 NGINX deployments를 예약합니다
```sh
kubernetes_deployment.nginx: Creating...
kubernetes_deployment.nginx: Still creating... [10s elapsed]
kubernetes_deployment.nginx: Still creating... [20s elapsed]
kubernetes_deployment.nginx: Still creating... [30s elapsed]
kubernetes_deployment.nginx: Still creating... [40s elapsed]
kubernetes_deployment.nginx: Still creating... [50s elapsed]
kubernetes_deployment.nginx: Creation complete after 56s [id=default/scalable-nginx-example]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

적용이 완료되면 NGINX deployments가 실행 중인지 확인합니다.

```sh
❯ k get deploy | grep scalable-nginx-example
scalable-nginx-example          2/2     2            2           4m32s
```

# Schedule a Service
NGINX를 사용자에게 노출하는 데 사용할 수 있는 여러 [Kubernetes 서비스](https://kubernetes.io/docs/concepts/services-networking/service)가 있습니다.  

- nodePort
- LoadBalancer
Kubernetes 클러스터가 종류에 따라 로컬로 호스팅되는 경우 NodePort를 통해 NGINX 인스턴스를 노출하여 인스턴스에 액세스합니다. 
  
Kubernetes 클러스터가 클라우드 provider에서 호스팅되는 경우 LoadBalancer를 통해 NGINX 인스턴스를 노출하여 인스턴스에 액세스합니다. 이렇게 하면 클라우드 provider의 로드 밸런서를 사용하여 서비스를 외부에 노출합니다.  
  
Kubernetes Service 리소스 블록이 어떻게 Selector를 배포 Labels에 동적으로 할당하는지 확인하세요.
이렇게 하면 일치하지 않는 서비스 Label Selector로 인한 일반적인 버그를 피할 수 있습니다.

kubernetes.tf 파일에 다음 구성을 추가합니다. 이렇게 하면 node_port: 30201에서 NGINX 인스턴스가 노출됩니다.
```tf 
resource "kubernetes_service" "nginx" {
  metadata {
    name = "nginx-example"
  }
  spec {
    selector = {
      App = kubernetes_deployment.nginx.spec.0.template.0.metadata[0].labels.App
    }
    port {
      node_port   = 30201
      port        = 80
      target_port = 80
    }

    type = "NodePort"
  }
}
```
구성을 적용하여 NodePort 서비스를 스케줄하십시오

```sh
kubernetes_service.nginx: Creating...
kubernetes_service.nginx: Creation complete after 1s [id=default/nginx-example]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

적용이 완료되면 NGINX 서비스가 실행 중인지 확인합니다.
```sh
❯ k get svc | grep nginx
nginx-example                            NodePort    10.43.61.217    <none>        80:30201/TCP                                             19s
```

# Scale the deployment
Config에서 replicas 필드를 늘려 deployments를 확장할 수 있습니다. Kubernetes 배포의 복제본 수를 2에서 4로 변경합니다.

```sh
resource "kubernetes_deployment" "nginx" {
  ## ...

  spec {
    replicas = 4

    ## ...
  }

  ## ...
}
```

# Managing Custom Resources
기본 제공 리소스 및 데이터 소스 외에도 Terraform provider에는 사용자 지정 리소스 정의(CRD), 사용자 지정 리소스 또는 Terraform provider에 구축되지 않은 리소스를 관리할 수 있는 kubernetes_manifest 리소스도 포함되어 있습니다.

Terraform을 사용하여 CRD를 적용한 다음 사용자 지정 리소스를 관리합니다. 
다음 두 단계로 이 작업을 수행해야 합니다.  
  
1. 필요한 CRD를 클러스터에 적용  
2. 클러스터에 사용자 지정 리소스 적용
3. 
Plan 시 Terraform이 Kubernetes API를 쿼리하여 매니페스트 필드에 지정된 객체 종류에 대한 스키마를 확인하기 때문에 두 가지 적용 단계가 필요합니다. 
Terraform이 매니페스트에 정의된 리소스에 대한 CRD를 찾지 못하면 계획에서 오류를 반환합니다.

## Create a custom resource definition
crontab_crd.tf라는 새 파일을 만들고 cron 데이터를 CronTab이라는 리소스로 저장하도록 Kubernetes를 확장하는 CRD에 대한 다음 구성에 붙여넣습니다.
```tf
resource "kubernetes_manifest" "crontab_crd" {
  manifest = {
    "apiVersion" = "apiextensions.k8s.io/v1"
    "kind"       = "CustomResourceDefinition"
    "metadata" = {
      "name" = "crontabs.stable.example.com"
    }
    "spec" = {
      "group" = "stable.example.com"
      "names" = {
        "kind"   = "CronTab"
        "plural" = "crontabs"
        "shortNames" = [
          "ct",
        ]
        "singular" = "crontab"
      }
      "scope" = "Namespaced"
      "versions" = [
        {
          "name" = "v1"
          "schema" = {
            "openAPIV3Schema" = {
              "properties" = {
                "spec" = {
                  "properties" = {
                    "cronSpec" = {
                      "type" = "string"
                    }
                    "image" = {
                      "type" = "string"
                    }
                  }
                  "type" = "object"
                }
              }
              "type" = "object"
            }
          }
          "served"  = true
          "storage" = true
        },
      ]
    }
  }
}

```
리소스에는 cronSpec 및 image라는 두 개의 구성 가능한 필드가 있습니다. 구성을 적용하여 CRD를 만듭니다. 예를 선택하여 신청을 확인하십시오.

Plan에서 Terraform은 두 가지 속성(매니페스트 및 개체)이 있는 리소스를 생성했습니다.  
  
매니페스트 속성은 원하는 구성이고 객체는 Terraform이 리소스를 생성한 후 Kubernetes API 서버에서 반환한 최종 상태입니다.  

Terraform이 Kubernetes API 서버가 추가할 수 있는 가능한 모든 리소스 속성을 포함하는 스키마를 생성했기 때문에 개체 속성에는 매니페스트에 지정한 것보다 더 많은 필드가 포함되어 있습니다. 출력 또는 다른 리소스에서 kubernetes_manifest 리소스를 참조할 때는 항상 object 속성을 사용하세요.

- 조회
```sh
❯ kubectl get crds crontabs.stable.example.com
NAME                          CREATED AT
crontabs.stable.example.com   2023-02-08T04:47:52Z
```

- 이미지
![[Pasted image 20230208134627.png]]



## Create a custom resource
이제 my_new_crontab.tf라는 새 파일을 만들고 새로 만든 CronTab CRD를 기반으로 사용자 지정 리소스를 만드는 다음 구성에 붙여넣습니다.
```tf
resource "kubernetes_manifest" "my_new_crontab" {
  manifest = {
    "apiVersion" = "stable.example.com/v1"
    "kind"       = "CronTab"
    "metadata" = {
      "name"      = "my-new-cron-object"
      "namespace" = "default"
    }
    "spec" = {
      "cronSpec" = "* * * * */5"
      "image"    = "my-awesome-cron-image"
    }
  }
}

```
구성을 적용하여 사용자 지정 리소스를 만듭니다. 예를 선택하여 적용을 확인합니다.

Terraform이 사용자 지정 리소스를 생성했는지 확인합니다.
```sh
❯ k get crontab
NAME                 AGE
my-new-cron-object   2s
```

```sh
❯ k describe crontab
Name:         my-new-cron-object
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  stable.example.com/v1
Kind:         CronTab
Metadata:
  Creation Timestamp:  2023-02-08T04:50:59Z
  Generation:          1
  Managed Fields:
    API Version:  stable.example.com/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        f:cronSpec:
        f:image:
    Manager:         Terraform
    Operation:       Apply
    Time:            2023-02-08T04:50:59Z
  Resource Version:  35072834
  UID:               1d124d76-c426-4758-86c7-b5fb339bd029
Spec:
  Cron Spec:  * * * * */5
  Image:      my-awesome-cron-image
Events:       <none>
```


# Clean up your workspace
이 튜토리얼을 마치면 생성한 모든 리소스를 폐기하십시오.  
  
terraform destroy를 실행하면 이 튜토리얼에서 생성한 NGINX 배포 및 서비스가 프로비저닝 해제됩니다. 예를 선택하여 파괴를 확인합니다.