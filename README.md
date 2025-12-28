# Домашнее задание к занятию «Отказоустойчивость в облаке»

## Задание 1 

Возьмите за основу [решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке»](https://github.com/netology-code/sdvps-homeworks/blob/main/7-03.md#задание-1).

1. Теперь вместо одной виртуальной машины сделайте terraform playbook, который:

- создаст 2 идентичные виртуальные машины. Используйте аргумент [count](https://www.terraform.io/docs/language/meta-arguments/count.html) для создания таких ресурсов;
- создаст [таргет-группу](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_target_group). Поместите в неё созданные на шаге 1 виртуальные машины;
- создаст [сетевой балансировщик нагрузки](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/lb_network_load_balancer), который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.

Рекомендуем изучить [документацию сетевого балансировщика нагрузки](https://cloud.yandex.ru/docs/network-load-balancer/quickstart) для того, чтобы было понятно, что вы сделали.

2. Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

3. Перейдите в веб-консоль Yandex Cloud и убедитесь, что: 

- созданный балансировщик находится в статусе Active,
- обе виртуальные машины в целевой группе находятся в состоянии healthy.

4. Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.

*В качестве результата пришлите:*

*1. Terraform Playbook.*

*2. Скриншот статуса балансировщика и целевой группы.*

*3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.*

## Решение 1

1. Терраформ плейбуки:

main.tf
```HCL
data "yandex_compute_image" "ubuntu_2404_lts" {
  family = "ubuntu-2404-lts"
}

resource "yandex_compute_instance" "vm-sflt-51-mia" {
  count       = 2 # указываем количество создаваемых ВМ
  name        = "vm-sflt-51-mia-${count.index}"
  hostname    = "vm-sflt-51-mia-${count.index}"
  platform_id = "standard-v4a"
  zone        = "ru-central1-a"

  resources {
    cores         = var.test.cores
    memory        = var.test.memory
    core_fraction = var.test.core_fraction
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu_2404_lts.image_id
      type     = "network-hdd"
      size     = 10
    }
  }

  metadata = {
    user-data          = file("./cloud-init.yml")
    serial-port-enable = 1
  }

  scheduling_policy { preemptible = true }

  network_interface {
    subnet_id          = yandex_vpc_subnet.sflt-51-mia_a.id
    nat                = true
  }
}



resource "local_file" "inventory" { # записываем ip-адреса машин для установки nginx через Ansible
  content  = <<-XYZ
    [webservers]
  ${yandex_compute_instance.vm-sflt-51-mia[0].network_interface.0.ip_address}
  ${yandex_compute_instance.vm-sflt-51-mia[1].network_interface.0.ip_address}

  XYZ
  filename = "./hosts.ini"
}
```

networks.tf
```HCL
resource "yandex_vpc_network" "sflt-51-mia" {
  name = "${var.flow}"
}

resource "yandex_vpc_subnet" "sflt-51-mia_a" {
  name           = "${var.flow}-ru-central1-a"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.sflt-51-mia.id
  v4_cidr_blocks = ["10.0.1.0/24"]
}
```

lb.tf
```HCL
resource "yandex_lb_target_group" "sflt-51-mia_tg" {
  name      = "${var.flow}-target-group"
  region_id = "ru-central1"

  dynamic "target" {
    for_each = yandex_compute_instance.vm-sflt-51-mia
    content {
      subnet_id = yandex_vpc_subnet.sflt-51-mia_a.id
      address   = target.value.network_interface.0.ip_address
    }
  }
}

resource "yandex_lb_network_load_balancer" "sflt-51-mia_nlb" {
  name = "${var.flow}-load-balancer"

  listener {
    name = "${var.flow}-listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.sflt-51-mia_tg.id

    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}
```

variables.tf
```HCL
variable "flow" {
  type    = string
  default = "sflt-51-mia"
}

variable "cloud_id" {
  type    = string
  default = "b1geap6dpsnun6sh70qj"
}
variable "folder_id" {
  type    = string
  default = "b1gfjo6em0ve76o982vg"
}

variable "test" {
  type = map(number)
  default = {
    cores         = 2
    memory        = 1
    core_fraction = 20
  }
}
```

Ansible-playbook для установки nginx
```YAML
---
- hosts: "webservers"
  vars:
    ansible_ssh_user: user
  become: true

  pre_tasks:
    - name: Validating the ssh port is open
      wait_for:
        host: "{{ (ansible_ssh_host|default(ansible_host))|default(inventory_hostname) }}"
        port: 22
        delay: 5
        timeout: 300
        state: started
        search_regex: OpenSSH

  tasks:
  - name: "Install nginx"
    apt:
      name: nginx
      state: present
      update_cache: yes
  - name: "Start nginx"
    service:
      name: nginx
      state: started
```

2. Скриншоты статуса балансировщика и целевой группы:

![Целевая группа]()
![Статус балансировщика и состояния ВМ в целевой группе]()

3. Скриншот страницы при запросе IP-адреса балансировщика:

![Стартовая страница nginx по адресу балансировщика]()

---

## Задание 2

1. Теперь вместо создания виртуальных машин создайте [группу виртуальных машин с балансировщиком нагрузки](https://cloud.yandex.ru/docs/compute/operations/instance-groups/create-with-balancer).

2. Nginx нужно будет поставить тоже автоматизированно. Для этого вам нужно будет подложить файл установки Nginx в user-data-ключ [метадаты](https://cloud.yandex.ru/docs/compute/concepts/vm-metadata) виртуальной машины.

- [Пример файла установки Nginx](https://github.com/nar3k/yc-public-tasks/blob/master/terraform/metadata.yaml).
- [Как подставлять файл в метадату виртуальной машины.](https://github.com/nar3k/yc-public-tasks/blob/a6c50a5e1d82f27e6d7f3897972adb872299f14a/terraform/main.tf#L38)

3. Перейдите в веб-консоль Yandex Cloud и убедитесь, что: 

- созданный балансировщик находится в статусе Active,
- обе виртуальные машины в целевой группе находятся в состоянии healthy.

4. Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.

*В качестве результата пришлите*

*1. Terraform Playbook.*

*2. Скриншот статуса балансировщика и целевой группы.*

*3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.*

## Решение 2

1. Terraform плейбук:

```HCL
data "yandex_compute_image" "ubuntu_2404_lts" {
  family = "ubuntu-2404-lts"
}

resource "yandex_compute_instance_group" "ig-sflt-51-mia" {
  name        = "ig-sflt-51-mia"
  folder_id           = var.folder_id
  service_account_id  = "ajemrjf2h6kj8fvg2vov"
  deletion_protection = "false"
  
  instance_template {
    platform_id = "standard-v4a"

    resources {
      cores         = var.test.cores
      memory        = var.test.memory
      core_fraction = var.test.core_fraction
    }

    boot_disk {
      initialize_params {
        image_id = data.yandex_compute_image.ubuntu_2404_lts.image_id
        type     = "network-hdd"
        size     = 10
      }
    }

    metadata = {
      user-data          = file("./cloud-init.yml")
      serial-port-enable = 1
    }

    scheduling_policy { preemptible = true }

    network_interface {
      network_id  = "${yandex_vpc_network.sflt-51-mia.id}"
      subnet_ids  = ["${yandex_vpc_subnet.sflt-51-mia_a.id}"]
      nat         = true
    }
  }

  scale_policy {
    fixed_scale {
      size = 2
    }
  }

  allocation_policy {
    zones = ["ru-central1-a"]
  }

  deploy_policy {
    max_unavailable = 1
    max_expansion   = 0
  }

  load_balancer {
    target_group_name = "sflt51-mia-tg"
    target_group_description = "Target group for Network Load Balancer"
  }
}

resource "yandex_lb_network_load_balancer" "sflt51-mia-lb-1" {
  name = "${var.flow}-lb-1"

  listener {
    name = "${var.flow}-listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_compute_instance_group.ig-sflt-51-mia.load_balancer.0.target_group_id

    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

resource "yandex_vpc_network" "sflt-51-mia" {
  name = "${var.flow}"
}

resource "yandex_vpc_subnet" "sflt-51-mia_a" {
  name           = "${var.flow}-ru-central1-a"
  zone           = "ru-central1-a"
  network_id     = "${yandex_vpc_network.sflt-51-mia.id}"
  v4_cidr_blocks = ["10.0.1.0/24"]
}
```

Файл с метаданными:

```HCL
#cloud-config
users:
  - name: user
    groups: sudo
    shell: /bin/bash
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    ssh_authorized_keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILSToR3JR5gmXkTtgSvCezhJDOxkcG2H02REWf1WZye7 ilyas-murtazin@mia-vb
timezone: Europe/Moscow
package_update: true
package_upgrade: true
apt:
  preserve_sources_list: true
packages:
  - nginx
runcmd:
  - [ systemctl, reload, nginx.service ]
  - [ systemctl, enable, nginx.service ]
  - [ systemctl, start, --no-block, nginx.service ]
  - [ sh, -c, "echo $(hostname | cut -d '.' -f 1 ) > /var/www/html/index.html" ]
```

2. Скриншоты статуса балансировщика и целевой группы:

![Целевая группа задание 2]()
![Статус балансировщика и состояния ВМ в целевой группе задание 2]()

3. Скриншот страницы при запросе IP-адреса балансировщика:

![Стартовая страница nginx по адресу балансировщика задание 2]()
![Результат нескольких обращений через curl на адрес балансировщика]()
