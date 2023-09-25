# Terraform4
Terraform - youtube

Terraform за 25 минут

default.tf
```tf
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

locals {
  cloud_id  = "b1gmtujeraulvnf2bj1i"
  folder_id = "b1ghauke2h8p27vt648a"
}

provider "yandex" {
    cloud_id = local.cloud_id
    folder_id = local.folder_id
    service_account_key_file = "/Users/joos/Downloads/authorized_key.json"
}
```

main.tf
```tf
locals {
  bucket_name = "tf-intro-site-bucket-1"
  index = "index.html"
}

resource "yandex_iam_service_account" "sa" {
  folder_id = local.folder_id
  name = "tf-test-sa"
}

// Назначение роли сервисному аккаунту
resource "yandex_resourcemanager_folder_iam_member" "sa-editor" {
  folder_id = local.folder_id
  role      = "storage.admin"
  member    = "serviceAccount:${yandex_iam_service_account.sa.id}"
}

// Создание статического ключа доступа
resource "yandex_iam_service_account_static_access_key" "sa-static-key" {
  service_account_id = yandex_iam_service_account.sa.id
  description        = "static access key for object storage"
}

// Создание бакета с использованием ключа
resource "yandex_storage_bucket" "test" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  bucket     = local.bucket_name
  acl = "public-read"

  website {
    index_document = local.index
  }
}

// ресурс странички
resource "yandex_storage_object" "index" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  acl = "public-read"
  bucket     = yandex_storage_bucket.test.id
  key        = local.index
  #source     = "site/${local.index}"
  content_base64 = base64encode(local.index_template)
  content_type = "text/html"
}

// ресурс картинки
resource "yandex_storage_object" "img" {
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  acl        = "public-read"
  bucket     = yandex_storage_bucket.test.id
  key        = each.key
  source     = "site/${each.key}"
  for_each   = fileset("site", "img/*")
}

// создаем переменную шаблон и передаем туда переменную
locals {
  index_template = templatefile("site/${local.index}.tpl", {
    endpoint = yandex_storage_bucket.test.website_endpoint
  })
}

output "site_name" {
  value = yandex_storage_bucket.test.website_endpoint
}
```
index.html -> index.html.tpl
```html
<html>
<head>
    <title>My first page</title>
</head>
<body>
    <h1>Hello Cats!</h1>
    
    <p><img src="img/cat1.jpg"></p>

    Site endpoint ${endpoint}

</body>

</html>
```


**DevOps с Nixys | Знакомство с Terraform**

terraform apply -auto-approve - создание с автоподтверждением

terraform fmt - автоформатирование

terraform destroy -target yandex_compute_instance.nixys - уничтожение только определенного инстанса

.tf - файлы конфигурации

.tfvars - файлы переменных

.tfstate - файлы текущего состояния инфраструктуры

.terraform - скаченные провайдеры, указанный в конфигурации

.terraform.lock.hcl - описываются зависимости модулей и провайдеров

main.tf - пример поднятия веб сервера через скрипт
```tf
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"

    // сохраняем terraform1.tfstate на сервере
    backend "s3" {
    endpoint   = "storage.yandexcloud.net"
    bucket     = "test-joos"
    region     = "ru-central1"
    key        = "terraform1.tfstate"
    shared_credentials_file = "storage.key"

    skip_region_validation      = true
    skip_credentials_validation = true
  }
}

provider "yandex" {
  service_account_key_file = "/Users/joos/MyTerraform/25min/authorized_key.json"
  cloud_id                 = "b1gmtujeraulvnf2bj1i"
  folder_id                = "b1ghauke2h8p27vt648a"
  zone                     = "ru-central1-b"
}

//сеть
resource "yandex_vpc_network" "test-vpc" {
  name = "nixys"
}

//подсеть
resource "yandex_vpc_subnet" "test-subnet" {
  v4_cidr_blocks = ["10.2.0.0/16"]
  network_id     = "${yandex_vpc_network.test-vpc.id}"
}

// создаем группу безопасности
resource "yandex_vpc_security_group" "test-sg" {
  name        = "Test security group"
  description = "Description for security group"
  network_id  = "${yandex_vpc_network.test-vpc.id}"

  dynamic "ingress" {
    for_each = ["80", "8080"]
    content {
      protocol       = "TCP"
      description    = "Rule description 1"
      v4_cidr_blocks = ["0.0.0.0/0"]
      from_port      = ingress.value
      to_port        = ingress.value
    }
  }

  egress {
    protocol       = "ANY"
    description    = "Rule description 2"
    v4_cidr_blocks = ["0.0.0.0/0"]
    from_port      = 0
    to_port        = 65535
  }
}

// делаем статический адрес
resource "yandex_vpc_address" "test-ip" { 
  name = "justAdress"
  external_ipv4_address {
    zone_id = "ru-central1-b"
  }  
}

// создаем компьютер
resource "yandex_compute_instance" "nixys" {
  name  = "nixys"
  platform_id = "standard-v3"

  resources {
  core_fraction = 20
  cores     = 2
  memory    = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd81ojtctf7kjqa3au3i"
    }
  }
  network_interface {
    subnet_id = yandex_vpc_subnet.test-subnet.id
    nat       = true
    nat_ip_address = yandex_vpc_address.test-ip.external_ipv4_address.0.address
  }
  
  // для подключения пользователя и исполняемый файл
  metadata = {
    ssh-keys = "debian:${file("/Users/joos/.ssh/id_rsa.pub")}"
    user-data = "${file("./init.sh")}"
  }
}

// выводы
output "external_ip" {
  value = yandex_vpc_address.test-ip.external_ipv4_address.0.address
}

output "external_ip2" {
  value = yandex_compute_instance.nixys.network_interface.0.nat_ip_address
}
```
init.sh
```sh
#!/bin/bash

### Install

apt-get update && apt-get install -y apt-transport-https ca-certificates curl software-properties-common nginx
echo "<h2>Joos</h2>" > /var/www/html/index.html
sudo service nginx start
```
storage.key  - сервисные аккаунты - аккаунт - создать новый статический ключ
```key
[default]
  aws_access_key_id = user.key1
  aws_secret_access_key = user.key2
```

main.tf - пример кластера с nginx с автомасштабированием
```tf
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }

    // сохраняем terraform1.tfstate на сервере
    backend "s3" {
    endpoint   = "storage.yandexcloud.net"
    bucket     = "test-joos"
    region     = "ru-central1"
    key        = "terraform_kub.tfstate"
    shared_credentials_file = "storage.key"

    skip_region_validation      = true
    skip_credentials_validation = true
  }
}

provider "yandex" {
  service_account_key_file = "/Users/joos/MyTerraform/25min/authorized_key.json"
  cloud_id                 = "b1gmtujeraulvnf2bj1i"
  folder_id                = "b1ghauke2h8p27vt648a"
  zone                     = "ru-central1-b"
}

# Переменные
locals {
  cloud_id    = "b1gmtujeraulvnf2bj1i"
  folder_id   = "b1ghauke2h8p27vt648a"
  k8s_version = "1.24"
  sa_name     = "myaccount"
}
# Сеть
resource "yandex_vpc_network" "mynet" {
  name = "mynet"
}
# Подсети
resource "yandex_vpc_subnet" "mysubnet-a" {
  v4_cidr_blocks = ["10.5.0.0/16"]
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.mynet.id
}

resource "yandex_vpc_subnet" "mysubnet-b" {
  v4_cidr_blocks = ["10.6.0.0/16"]
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.mynet.id
}

resource "yandex_vpc_subnet" "mysubnet-c" {
  v4_cidr_blocks = ["10.7.0.0/16"]
  zone           = "ru-central1-c"
  network_id     = yandex_vpc_network.mynet.id
}
# Группы безопасности
resource "yandex_vpc_security_group" "k8s-main-sg" {
  name        = "k8s-main-sg"
  description = "Правила группы обеспечивают базовую работоспособность кластера Managed Service for Kubernetes. Примените ее к кластеру Managed Service for Kubernetes и группам узлов."
  network_id  = yandex_vpc_network.mynet.id
  ingress {
    protocol          = "TCP"
    description       = "Правило разрешает проверки доступности с диапазона адресов балансировщика нагрузки. Нужно для работы отказоустойчивого кластера Managed Service for Kubernetes и сервисов балансировщика."
    predefined_target = "loadbalancer_healthchecks"
    from_port         = 0
    to_port           = 65535
  }
  ingress {
    protocol          = "ANY"
    description       = "Правило разрешает взаимодействие мастер-узел и узел-узел внутри группы безопасности."
    predefined_target = "self_security_group"
    from_port         = 0
    to_port           = 65535
  }
  ingress {
    protocol          = "ANY"
    description       = "Правило разрешает взаимодействие под-под и сервис-сервис. Укажите подсети вашего кластера Managed Service for Kubernetes и сервисов."
    v4_cidr_blocks    = concat(yandex_vpc_subnet.mysubnet-a.v4_cidr_blocks, yandex_vpc_subnet.mysubnet-b.v4_cidr_blocks, yandex_vpc_subnet.mysubnet-c.v4_cidr_blocks)
    from_port         = 0
    to_port           = 65535
  }
  ingress {
    protocol          = "ICMP"
    description       = "Правило разрешает отладочные ICMP-пакеты из внутренних подсетей."
    v4_cidr_blocks    = ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
  }
  ingress {
    protocol          = "TCP"
    description       = "Правило разрешает входящий трафик из интернета на диапазон портов NodePort. Добавьте или измените порты на нужные вам."
    v4_cidr_blocks    = ["0.0.0.0/0"]
    from_port         = 30000
    to_port           = 32767
  }
    ingress {
    protocol          = "TCP"
    description       = "Правило разрешает входящий трафик из интернета на диапазон портов NodePort. Добавьте или измените порты на нужные вам."
    v4_cidr_blocks    = ["0.0.0.0/0"]
    port         = 443
  }
  egress {
    protocol          = "ANY"
    description       = "Правило разрешает весь исходящий трафик. Узлы могут связаться с Yandex Container Registry, Yandex Object Storage, Docker Hub и т. д."
    v4_cidr_blocks    = ["0.0.0.0/0"]
    from_port         = 0
    to_port           = 65535
  }
}
# Создание сервисного аккаунта
resource "yandex_iam_service_account" "myaccount" {
  name        = local.sa_name
  description = "K8S regional service account"
}
# Назначение ролей
resource "yandex_resourcemanager_folder_iam_member" "editor" {
  # Сервисному аккаунту назначается роль "editor".
  folder_id = local.folder_id
  role      = "admin"
  member    = "serviceAccount:${yandex_iam_service_account.myaccount.id}"
}

resource "yandex_resourcemanager_folder_iam_member" "images-puller" {
  # Сервисному аккаунту назначается роль "container-registry.images.puller".
  folder_id = local.folder_id
  role      = "container-registry.images.puller"
  member    = "serviceAccount:${yandex_iam_service_account.myaccount.id}"
}

resource "yandex_kms_symmetric_key" "kms-key" {
  # Ключ для шифрования важной информации, такой как пароли, OAuth-токены и SSH-ключи.
  name              = "kms-key"
  default_algorithm = "AES_128"
  rotation_period   = "8760h" # 1 год.
}

resource "yandex_resourcemanager_folder_iam_member" "viewer" {
  folder_id = local.folder_id
  role      = "viewer"
  member    = "serviceAccount:${yandex_iam_service_account.myaccount.id}"
}

# Создание кластера
resource "yandex_kubernetes_cluster" "k8s-regional" {
  network_id = yandex_vpc_network.mynet.id
  network_policy_provider = "CALICO"
  master {
    version = local.k8s_version
    public_ip = true
    regional {
      region = "ru-central1"
      # задействуем 3 подсети
      location {
        zone      = yandex_vpc_subnet.mysubnet-a.zone
        subnet_id = yandex_vpc_subnet.mysubnet-a.id
      }
      location {
        zone      = yandex_vpc_subnet.mysubnet-b.zone
        subnet_id = yandex_vpc_subnet.mysubnet-b.id
      }
      location {
        zone      = yandex_vpc_subnet.mysubnet-c.zone
        subnet_id = yandex_vpc_subnet.mysubnet-c.id
      }
    }
    security_group_ids = [yandex_vpc_security_group.k8s-main-sg.id]
  }# Используем сервисные аккаунты
  service_account_id      = yandex_iam_service_account.myaccount.id
  node_service_account_id = yandex_iam_service_account.myaccount.id
  depends_on = [# Указываем последовательность создания ресурсов
    yandex_resourcemanager_folder_iam_member.editor,
    yandex_resourcemanager_folder_iam_member.images-puller
  ]
  kms_provider {
    key_id = yandex_kms_symmetric_key.kms-key.id
  }
}

# Создаем группы нод
resource "yandex_kubernetes_node_group" "my_node_group_a" {
  cluster_id  = "${yandex_kubernetes_cluster.k8s-regional.id}"
  name        = "worker-a"
  description = "description"
  version     = local.k8s_version

  labels = {
    "key" = "value"
  }

  instance_template {
    platform_id = "standard-v3"
    name = "worker-a-{instance.short_id}"

    network_interface {
      nat                = true
      subnet_ids         = [yandex_vpc_subnet.mysubnet-a.id]
      security_group_ids = [yandex_vpc_security_group.k8s-main-sg.id]
    }

    resources {
      memory = 2
      cores  = 2
    }

    boot_disk {
      type = "network-hdd"
      size = 20
    }

    scheduling_policy {
      preemptible = false
    }

  }

  scale_policy {
    auto_scale {
      min     = 1
      max     = 3
      initial = 1
    }
  }

  allocation_policy {
    location {
      zone = "ru-central1-a"
    }
  }
}

resource "yandex_kubernetes_node_group" "my_node_group_b" {
  cluster_id  = "${yandex_kubernetes_cluster.k8s-regional.id}"
  name        = "worker-b"
  description = "description"
  version     = local.k8s_version

  labels = {
    "key" = "value"
  }

  instance_template {
    platform_id = "standard-v3"
    name = "worker-b-{instance.short_id}"

    network_interface {
      nat                = true
      subnet_ids         = [yandex_vpc_subnet.mysubnet-b.id]
      security_group_ids = [yandex_vpc_security_group.k8s-main-sg.id]
    }

    resources {
      memory = 2
      cores  = 2
    }

    boot_disk {
      type = "network-hdd"
      size = 32
    }

    scheduling_policy {
      preemptible = false
    }

  }

  scale_policy {
    auto_scale {
      min     = 1
      max     = 3
      initial = 1
    }
  }

  allocation_policy {
    location {
      zone = "ru-central1-b"
    }
  }
}

resource "yandex_kubernetes_node_group" "my_node_group_c" {
  cluster_id  = "${yandex_kubernetes_cluster.k8s-regional.id}"
  name        = "worker-c"
  description = "description"
  version     = local.k8s_version

  labels = {
    "key" = "value"
  }

  instance_template {
    platform_id = "standard-v3"
    name = "worker-c-{instance.short_id}"

    network_interface {
      nat                = true
      subnet_ids         = [yandex_vpc_subnet.mysubnet-c.id]
      security_group_ids = [yandex_vpc_security_group.k8s-main-sg.id]
    }

    resources {
      memory = 2
      cores  = 2
    }

    boot_disk {
      type = "network-hdd"
      size = 32
    }

    scheduling_policy {
      preemptible = false
    }

  }

  scale_policy {
    auto_scale {
      min     = 1
      max     = 3
      initial = 1
    }
  }

  allocation_policy {
    location {
      zone = "ru-central1-c"
    }
  }
}
```
test_autoscaling.yml
```yml
---
### Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: registry.k8s.io/hpa-example
          resources:
            requests:
              memory: "256Mi"
              cpu: "500m"
            limits:
              memory: "500Mi"
              cpu: "1"
---
### Service
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
---
### HPA
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 20
```
