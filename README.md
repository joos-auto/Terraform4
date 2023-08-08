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
