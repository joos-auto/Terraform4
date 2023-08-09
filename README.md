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

main.tf
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

