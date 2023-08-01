
```hcl
# Объявление переменных для пользовательских параметров

variable "zone" {
  type = string
}

variable "folder_id" {
  type = string
}

variable "vm_image_family" {
  type = string
}

variable "vm_user" {
  type = string
}

variable "ssh_key_path" {
  type = string
}

variable "dns_zone" {
  type = string
}

# Добавление прочих переменных

locals {
  network_name       = "web-network"
  subnet_name        = "subnet1"
  sg_vm_name         = "sg-web"
  vm_name            = "lamp-vm"
  dns_zone_name      = "example-zone"
}

# Настройка провайдера

terraform {
  required_providers {
    yandex    = {
      source  = "yandex-cloud/yandex"
      version = ">= 0.47.0"
    }
  }
}

provider "yandex" {
  folder_id = var.folder_id
}

# Создание облачной сети

resource "yandex_vpc_network" "network-1" {
  name = local.network_name
}

# Создание подсети

resource "yandex_vpc_subnet" "subnet-1" {
  name           = local.subnet_name
  v4_cidr_blocks = ["192.168.1.0/24"]
  zone           = var.zone
  network_id     = yandex_vpc_network.network-1.id
}

# Создание группы безопасности

resource "yandex_vpc_security_group" "sg-1" {
  name        = local.sg_vm_name
  network_id  = yandex_vpc_network.network-1.id
  egress {
    protocol       = "ANY"
    description    = "any"
    v4_cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    protocol       = "TCP"
    description    = "ext-http"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 80
  }
  ingress {
    protocol       = "TCP"
    description    = "ext-https"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 443
  }
}

# Добавление готового образа ВМ

resource "yandex_compute_image" "lamp-vm-image" {
  source_family = var.vm_image_family
}

# Создание ВМ

resource "yandex_compute_instance" "vm-lamp" {
  name        = local.vm_name
  platform_id = "standard-v3"
  zone        = var.zone
  resources {
    core_fraction = 20
    cores         = 2
    memory        = 1
  }
  boot_disk {
    initialize_params {
      image_id = yandex_compute_image.lamp-vm-image.id
    }
  }
  network_interface {
    subnet_id          = yandex_vpc_subnet.subnet-1.id
    nat                = true
    security_group_ids = [yandex_vpc_security_group.sg-1.id]
  }
  metadata = {
    user-data = "#cloud-config\nusers:\n  - name: ${var.vm_user}\n    groups: sudo\n    shell: /bin/bash\n    sudo: 'ALL=(ALL) NOPASSWD:ALL'\n    ssh-authorized-keys:\n      - ${file("${var.ssh_key_path}")}"
  }
}

# Создание зоны DNS

resource "yandex_dns_zone" "zone1" {
  name    = local.dns_zone_name
  zone    = var.dns_zone
  public  = true
}

# Создание ресурсной записи типа А

resource "yandex_dns_recordset" "rs-a" {
  zone_id = yandex_dns_zone.zone1.id
  name    = var.dns_zone
  type    = "A"
  ttl     = 600
  data    = [ yandex_compute_instance.vm-lamp.network_interface.0.nat_ip_address ]
}

# Создание ресурсной записи типа CNAME

resource "yandex_dns_recordset" "rs-cname" {
  zone_id = yandex_dns_zone.zone1.id
  name    = "www"
  type    = "CNAME"
  ttl     = 600
  data    = [ var.dns_zone ]
}
```


