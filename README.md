# Домашнее задание к занятию «Установка Kubernetes»

### Цель задания

Установить кластер K8s.

### Чеклист готовности к домашнему заданию

1. Развёрнутые ВМ с ОС Ubuntu 20.04-lts.


### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция по установке kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).
2. [Документация kubespray](https://kubespray.io/).

-----

### Задание 1. Установить кластер k8s с 1 master node

1. Подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды.
2. В качестве CRI — containerd.
3. Запуск etcd производить на мастере.
4. Способ установки выбрать самостоятельно.

Для создания инфраструктуры написал pipeline для Terraform, столкнулся с кучей ошибок и проблем,на решение которых ушло не мало времени, так как в лекции совсем не уделено внимание инфраструктуре.
Создаются три мастер ноды с учетом следующего задания, 4 воркера и дополнительный инстанс для доступа к кластеру из вне и обаспечения доступа в Интернет нодам.
```hcl
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
#  required_version = ">= 0.13"
}

provider "yandex" {
  cloud_id                 = "b1g6dgftb02k9esf1nmu"
  folder_id                = "b1gpta86451pk7tseq2b"
  zone                     = "ru-central1-a" # Зона доступности по умолчанию
  service_account_key_file = file("~/key.json")
}


data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2004-lts" 
}
data "yandex_compute_image" "nat-instance-ubuntu" {
  family = "nat-instance-ubuntu"
}

resource "yandex_compute_instance" "bastion-nat" {
  
  name = "bastion-nat"
  allow_stopping_for_update = true
    resources {
    cores  = 2
    memory = 2
    core_fraction = 20
    
    }
    scheduling_policy {
    preemptible = true
    
    }
    boot_disk {

    initialize_params {
      image_id = data.yandex_compute_image.nat-instance-ubuntu.image_id
      size = 20
    }
    
    }
    network_interface {

    subnet_id          = yandex_vpc_subnet.k8s-out.id
    security_group_ids = [yandex_vpc_security_group.nat-instance-sg.id]
    nat                = true
    }
    metadata = {
    ssh-keys = "ubuntu:${file("~/.ssh/id_ed25519.pub")}"
  }
}
resource "yandex_compute_instance" "vm-1" {
  count = 3
  name = "control-node-${count.index + 1}"
    resources {
    cores  = 2
    memory = 2
    core_fraction = 20

    }
    scheduling_policy {
    preemptible = true
    }
    boot_disk {

    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.image_id
      size = 20
    }
    
    }
    network_interface {

    subnet_id          = yandex_vpc_subnet.k8s-subnet.id
    security_group_ids = [yandex_vpc_security_group.nat-instance-sg.id]
    nat                = false
    }
    metadata = {
    ssh-keys = "ubuntu:${file("~/.ssh/id_ed25519.pub")}"
  }
}
resource "yandex_compute_instance" "vm-2" {
  count = 4
  name = "worker-node-${count.index + 1}"
    resources {
    cores  = 2
    memory = 2
    core_fraction = 20

    }
    scheduling_policy {
    preemptible = true
    }
    boot_disk {

    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.image_id
      size = 20
    }
    
    }
    network_interface {

    subnet_id          = yandex_vpc_subnet.k8s-subnet.id
    security_group_ids = [yandex_vpc_security_group.nat-instance-sg.id]
    nat                = false
    }
    metadata = {
    ssh-keys = "ubuntu:${file("~/.ssh/id_ed25519.pub")}"
  }
}

resource "yandex_vpc_network" "k8s" {
  name = "k8s"
}

resource "yandex_vpc_subnet" "k8s-subnet" {
  zone           = "ru-central1-a"
  name = "k8s-subnet"
  network_id     = "${yandex_vpc_network.k8s.id}"
  v4_cidr_blocks = ["192.168.0.0/24"]
  route_table_id = yandex_vpc_route_table.nat-instance-route.id
}
resource "yandex_vpc_subnet" "k8s-out" {
  zone           = "ru-central1-a"
  name = "k8s-out"
  network_id     = "${yandex_vpc_network.k8s.id}"
  v4_cidr_blocks = ["192.168.1.0/24"]

}
# Создание таблицы маршрутизации и статического маршрута

resource "yandex_vpc_route_table" "nat-instance-route" {
  name       = "nat-instance-route"
  network_id = "${yandex_vpc_network.k8s.id}"
  static_route {
    destination_prefix = "0.0.0.0/0"
    next_hop_address   = yandex_compute_instance.bastion-nat.network_interface.0.ip_address
  }
}

resource "yandex_vpc_security_group" "nat-instance-sg" {
  name       = "sg"
  network_id = yandex_vpc_network.k8s.id

  egress {
    protocol       = "ANY"
    description    = "any"
    v4_cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    protocol       = "ANY"
    description    = "any"
    v4_cidr_blocks = ["0.0.0.0/0"]
  }


}




```
Дополнительно cделал output, чтобы удобно было определять ip адреса узлов:
```hcl
output "all_vms" {
  description = "Information about the instances"
  value = {
    bastion-nat = [
         {  name = yandex_compute_instance.bastion-nat.name
            nat_ip_address = yandex_compute_instance.bastion-nat.network_interface[0].nat_ip_address
        }
        ],    
    control = [
      for instance in yandex_compute_instance.vm-1 : {
        name = instance.name
        ip_address = instance.network_interface[0].ip_address
        
      }
    ],

    worker = [
          for instance in yandex_compute_instance.vm-2 : {
        name = instance.name
        ip_address = instance.network_interface[0].ip_address       
        }
        ]
         
}
} 
```
Применил конфигурацию:
```bash
output "all_vms" {
  description = "Information about the instances"
  value = {
    bastion-nat = [
         {  name = yandex_compute_instance.bastion-nat.name
            nat_ip_address = yandex_compute_instance.bastion-nat.network_interface[0].nat_ip_address
        }
        ],    
    control = [
      for instance in yandex_compute_instance.vm-1 : {
        name = instance.name
        ip_address = instance.network_interface[0].ip_address
        
      }
    ],

    worker = [
          for instance in yandex_compute_instance.vm-2 : {
        name = instance.name
        ip_address = instance.network_interface[0].ip_address       
        }
        ]
         
}
}
```
Кластер разворачивал при помощи Kubespray.

Сконфигурировал файл инвентаря hosts.yaml:
```yaml
all:
  hosts:
    node1:
      ansible_host: 192.168.0.14
      ip: 192.168.0.14
      access_ip: 192.168.0.14
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_user: ubuntu
      ansible_ssh_common_args: -J ubuntu@130.193.37.217
    node2:
      ansible_host: 192.168.0.24
      ip: 192.168.0.24
      access_ip: 192.168.0.24
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_user: ubuntu
      ansible_ssh_common_args: -J ubuntu@130.193.37.217   
    node3:
      ansible_host: 192.168.0.10
      ip: 192.168.0.10
      access_ip: 192.168.0.10
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_user: ubuntu
      ansible_ssh_common_args: -J ubuntu@130.193.37.217
    node4:
      ansible_host: 192.168.0.7
      ip: 192.168.0.7
      access_ip: 192.168.0.7
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_user: ubuntu
      ansible_ssh_common_args: -J ubuntu@130.193.37.217 
    node5:
      ansible_host: 192.168.0.16
      ip: 192.168.0.16
      access_ip: 192.168.0.16
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_user: ubuntu
      ansible_ssh_common_args: -J ubuntu@130.193.37.217
    node6:
      ansible_host: 192.168.0.4
      ip: 192.168.0.4
      access_ip: 192.168.0.4
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_user: ubuntu
      ansible_ssh_common_args: -J ubuntu@130.193.37.217
    node7:
      ansible_host: 192.168.0.3
      ip: 192.168.0.3
      access_ip: 192.168.0.3
      ansible_ssh_private_key_file: ~/.ssh/id_ed25519
      ansible_user: ubuntu
      ansible_ssh_common_args: -J ubuntu@130.193.37.217
  # [bastion]
  #bastion ansible_host=x.x.x.x ansible_user=    
  children:
    kube_control_plane:
      hosts:
        node1:
        node2:
        node3:
    kube_node:
      hosts:
        node4:
        node5:
        node6:
        node7:

    etcd:
      hosts:
        node1:
        node2:
        node3:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```
Применил:

```bash
\ current cluster configuration]                         /
 --------------------------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||

ok: [node1] => {
    "changed": false,
    "msg": "All assertions passed"
}
Monday 25 November 2024  13:28:50 +0000 (0:00:00.245)       0:35:12.730 ******* 
Monday 25 November 2024  13:28:50 +0000 (0:00:00.028)       0:35:12.759 ******* 
Monday 25 November 2024  13:28:50 +0000 (0:00:00.030)       0:35:12.790 ******* 
 ____________
< PLAY RECAP >
 ------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||

node1                      : ok=702  changed=153  unreachable=0    failed=0    skipped=1083 rescued=0    ignored=6   
node2                      : ok=607  changed=138  unreachable=0    failed=0    skipped=978  rescued=0    ignored=3   
node3                      : ok=609  changed=139  unreachable=0    failed=0    skipped=976  rescued=0    ignored=3   
node4                      : ok=478  changed=93   unreachable=0    failed=0    skipped=687  rescued=0    ignored=1   
node5                      : ok=478  changed=93   unreachable=0    failed=0    skipped=684  rescued=0    ignored=1   
node6                      : ok=478  changed=93   unreachable=0    failed=0    skipped=684  rescued=0    ignored=1   
node7                      : ok=478  changed=93   unreachable=0    failed=0    skipped=684  rescued=0    ignored=1   

Monday 25 November 2024  13:28:50 +0000 (0:00:00.102)       0:35:12.893 ******* 
=============================================================================== 
kubernetes/preinstall : Install packages requirements ------------------------------------------------------------------------ 90.74s
etcd : Gen_certs | Write etcd member/admin and kube_control_plane client certs to other etcd nodes --------------------------- 84.89s
kubernetes-apps/ingress_controller/ingress_nginx : NGINX Ingress Controller | Create manifests ------------------------------- 49.59s
kubernetes-apps/ansible : Kubernetes Apps | Lay Down CoreDNS templates ------------------------------------------------------- 45.73s
network_plugin/calico : Wait for calico kubeconfig to be created ------------------------------------------------------------- 43.48s
kubernetes/preinstall : Preinstall | wait for the apiserver to be running ---------------------------------------------------- 37.58s
download : Download_container | Download image if required ------------------------------------------------------------------- 33.30s
kubernetes/control-plane : Kubeadm | Initialize first control plane node ----------------------------------------------------- 29.73s
network_plugin/calico : Calico | Create calico manifests --------------------------------------------------------------------- 26.23s
download : Download_container | Download image if required ------------------------------------------------------------------- 23.89s
download : Download_container | Download image if required ------------------------------------------------------------------- 22.86s
kubernetes/control-plane : Kubeadm | Create kubeadm config ------------------------------------------------------------------- 18.14s
container-engine/containerd : Containerd | Unpack containerd archive --------------------------------------------------------- 17.83s
policy_controller/calico : Create calico-kube-controllers manifests ---------------------------------------------------------- 17.12s
etcd : Gen_certs | Gather etcd member/admin and kube_control_plane client certs from first etcd node ------------------------- 17.03s
network_plugin/cni : CNI | Copy cni plugins ---------------------------------------------------------------------------------- 16.65s
kubernetes/control-plane : Joining control plane node to the cluster. -------------------------------------------------------- 16.25s
kubernetes-apps/ansible : Kubernetes Apps | Start Resources ------------------------------------------------------------------ 16.15s
kubernetes-apps/helm : Extract_file | Unpacking archive ---------------------------------------------------------------------- 15.57s
download : Download_container | Download image if required ------------------------------------------------------------------- 15.46s
(.venv) user@microk8s:~/kuber-homeworks-3.2/kubespray$ 
```
Настрииваю kubectl и проверяю кластер:
```bash
(.venv) user@microk8s:~/kuber-homeworks-3.2/kubespray$ ssh ubuntu@192.168.0.14 -J ubuntu@130.193.37.217
The authenticity of host '192.168.0.14 (<no hostip for proxy command>)' can't be established.
ED25519 key fingerprint is SHA256:2OI8hXHZrJGYjz1x/AmUrEm8e6OB+xz2ONVV16GjnC8.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.0.14' (ED25519) to the list of known hosts.
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-200-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
New release '22.04.5 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Mon Nov 25 12:54:15 2024 from 192.168.1.3
ubuntu@node1:~$ kubectl get node
E1125 13:31:01.171645   23569 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
E1125 13:31:01.173523   23569 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
E1125 13:31:01.174985   23569 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
E1125 13:31:01.176525   23569 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
E1125 13:31:01.178373   23569 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp 127.0.0.1:8080: connect: connection refused"
The connection to the server localhost:8080 was refused - did you specify the right host or port?
ubuntu@node1:~$ mkdir .kube
ubuntu@node1:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
ubuntu@node1:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
ubuntu@node1:~$ kubectl get nodes 
NAME    STATUS   ROLES           AGE   VERSION
node1   Ready    control-plane   16m   v1.31.1
node2   Ready    control-plane   15m   v1.31.1
node3   Ready    control-plane   15m   v1.31.1
node4   Ready    <none>          14m   v1.31.1
node5   Ready    <none>          14m   v1.31.1
node6   Ready    <none>          14m   v1.31.1
node7   Ready    <none>          14m   v1.31.1
```




```

------

## Дополнительные задания (со звёздочкой)
### Задание 2*. Установить HA кластер

1. Установить кластер в режиме HA.
2. Использовать нечётное количество Master-node.
3. Для cluster ip использовать keepalived или другой способ.

Столкнулся с тем что, на яндекс облаке не работает  keepalived. Предполагаю что из-за того что нормально не работает arp протокол между нодами.
