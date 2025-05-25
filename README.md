# Домашнее задание к занятию «Отказоустойчивость в облаке»

## ` Климов Дмитрий `

## Задание 1

Возьмите за основу решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке».

1. Теперь вместо одной виртуальной машины сделайте terraform playbook, который:
создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;
создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;
создаст сетевой балансировщик нагрузки, который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.
Рекомендуем изучить документацию сетевого балансировщика нагрузки для того, чтобы было понятно, что вы сделали.

2. Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

3. Перейдите в веб-консоль Yandex Cloud и убедитесь, что:
созданный балансировщик находится в статусе Active,
обе виртуальные машины в целевой группе находятся в состоянии healthy.

4. Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.
В качестве результата пришлите:

1. Terraform Playbook.

2. Скриншот статуса балансировщика и целевой группы.

3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.

## Ответ:

` meta.yaml `

```

#cloud-config
disable_root: true
timezone: Europe/Moscow
repo_update: true
apt:
  preserve_sources_list: true
packages:
 - nginx
runcmd:
 - [ systemctl, nginx-reload ]
 - [ systemctl, enable, nginx.service ]
 - [ systemctl, start, --no-block, nginx.service ]
users:
 - name: netology
   groups: sudo
   shell: /bin/bash
   sudo: ['ALL=(ALL) NOPASSWD:ALL']
   ssh-authorized-keys:
     - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCbiLEQVL+IUjKTiVaJzH3uRRhFCEYR9BWjmvy9B4PCQE

```

` main.tf `

```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  token     = "y0__xDQ9KDFBxjB3RMg7qmF1hKV8J_uA62xWRbSqXKi4nDxtzkZ1A"  # ТВОЙ ТОКЕН!
  cloud_id  = "b1gpe6vnl2rgqimj7eo2"  # ТВОЙ CLOUD ID!
  folder_id = "b1g1ts9lbj0hhs975go7"  # ТВОЙ FOLDER ID!
  zone      = "ru-central1-b"
}

resource "yandex_compute_instance" "vm" {
  count = 2
  name  = "vm${count.index}"
  hostname = "vm${count.index}"
  resources {
    cores         = 2
    memory        = 4
  }

  boot_disk {
    initialize_params {
      image_id = "fd8huqdhr65m771g1bka"
      size     = 30
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }

}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network-1.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}

resource "yandex_lb_target_group" "target-1" {
  name = "target-1"

  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[0].network_interface.0.ip_address
  }

  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[1].network_interface.0.ip_address
  }

}

resource "yandex_lb_network_load_balancer" "lb-1" {
  name = "lb1"

  listener {
    name                = "listener"
    port                = 80
    protocol            = "tcp"
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.target-1.id
    healthcheck {
      name = "http"
        http_options {
          port = 80
          path = "/"
        }
    }
  }
}

output "internal_ip_address_vm-0" {
  value = yandex_compute_instance.vm[0].network_interface.0.ip_address
}
output "external_ip_address_vm-0" {
  value = yandex_compute_instance.vm[0].network_interface.0.nat_ip_address
}
output "internal_ip_address_vm-1" {
  value = yandex_compute_instance.vm[1].network_interface.0.ip_address
}
output "external_ip_address_vm-1" {
  value = yandex_compute_instance.vm[1].network_interface.0.nat_ip_address
}

```

![Снимок экрана (1079)](https://github.com/user-attachments/assets/fef57c4a-fc7e-47e2-bf91-f1a86b413cda)

![Снимок экрана (1081)](https://github.com/user-attachments/assets/2c26bdf4-dbab-4be6-ac30-66657d4b5fd8)

![Снимок экрана (1082)](https://github.com/user-attachments/assets/9081e0f6-28d5-46c0-a2a8-b159e3ada3e5)

![Снимок экрана (1083)](https://github.com/user-attachments/assets/7bc71a42-d99d-4487-bff6-f2e9048aaf39)

![Снимок экрана (1084)](https://github.com/user-attachments/assets/2f7af615-a32b-4ff7-bdc5-6ddb8b53d204)




