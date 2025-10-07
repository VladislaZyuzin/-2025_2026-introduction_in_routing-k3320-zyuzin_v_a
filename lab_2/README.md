# Отчет по лабораторной работе №2

## Университет
* **University:** [ITMO University](https://itmo.ru/ru/)
* **Faculty:** [FICT](https://fict.itmo.ru)
* **Course:** [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)
* **Year:** 2025/2026
* **Group:** K3320
* **Author:** Zyuzin Vladislav Alexandrovich 
* **Lab:** Lab2
* **Date of create:** 06.10.2023
* **Date of finished:** 07.10.2023

## Задание

Вам необходимо сделать сеть связи в трех геораспределенных офисах "RogaIKopita Games" изображенную на рисунке ниже в ContainerLab. Необходимо создать все устройства указанные на схеме и соединения между ними.

* Помимо этого вам необходимо настроить IP адреса на интерфейсах.
* Создать DHCP сервера на роутерах в сторону клиентских устройств.
* Настроить статическую маршрутизацию.
* Настроить имена устройств, сменить логины и пароли.

![Схема из задания](<img width="541" height="361" alt="image" src="https://github.com/user-attachments/assets/99929a67-daae-4ab6-abfb-fafe552f93ed" />
)

## Описание работы

Для выполнения работы был взят стейдж сервер. На него были установлены `docker`, `make` и `containerlab`, а также склонирован репозиторий `hellt/vrnetlab` (в папку routeros был загружен файл chr-6.47.9.vmdk). C помощью `make docker-image` был собран соответствуший образ.

### Топология 
Файл `lab.yml` описывает топологию сети, состоящую из трёх маршрутизаторов (R01, R02, R03), представляющих три разных офиса компании, и трёх ПК (PC1, PC2, PC3), которые подключены к своим локальным маршрутизаторам. Маршрутизаторы соединены между собой линками, и каждый ПК подключён к своему маршрутизатору через локальный интерфейс.
```yaml
name: lab2

topology:
    kinds:
        vr-mikrotik_ros:
            image: vrnetlab/mikrotik_routeros:6.47.9
        linux:
            image: alpine:latest
    nodes:
        R01_Moscow:
            kind: vr-mikrotik_ros
            mgmt-ipv4: 192.168.50.10
            startup-config: ./configs/r1.rsc
        R02_Berlin:
            kind: vr-mikrotik_ros
            mgmt-ipv4: 192.168.50.20
            startup-config: ./configs/r2.rsc
        R03_Frankfurt:
            kind: vr-mikrotik_ros
            mgmt-ipv4: 192.168.50.30
            startup-config: ./configs/r3.rsc
        PC1:
            kind: linux
            binds:
              - ./configs:/configs
        PC2:
            kind: linux
            binds:
              - ./configs:/configs
        PC3:
            kind: linux
            binds:
              - ./configs:/configs
    links:
        - endpoints: ["R01_Moscow:eth2", "R02_Berlin:eth2"]
        - endpoints: ["R01_Moscow:eth3", "R03_Frankfurt:eth3"]
        - endpoints: ["R01_Moscow:eth4", "PC1:eth2"]
        - endpoints: ["R02_Berlin:eth3", "R03_Frankfurt:eth2"]
        - endpoints: ["R02_Berlin:eth4", "PC2:eth2"]
        - endpoints: ["R03_Frankfurt:eth4", "PC3:eth2"]

mgmt:
    network: mgmt-net
    ipv4-subnet: 192.168.50.0/24
```

Ниже можно ознакомиться с графическим представлением этой схемы:


<img width="562" height="510" alt="lab2-topology drawio" src="https://github.com/user-attachments/assets/a420fec7-a47f-49ac-9f20-d539cead097d" />


### Настройка маршрутизаторов
На маршрутизаторе R01 были настроены DHCP-сервер для выдачи IP-адресов клиентам в локальной сети, статические маршруты для взаимодействия с другими офисами, а также интерфейсы с соответствующими IP-адресами для каждого соединения. Имя устройства было изменено на R01_Moscow, и настроены основные параметры сети для связи с R02 и R03.

Пример настройки `R01`:
```rsc
/ip pool
add name=dhcp_pool_Moscow ranges=192.168.1.3-192.168.1.200
/ip dhcp-server
add address-pool=dhcp_pool_Moscow disabled=no interface=ether5 name=dhcp_Moscow
/ip address
add address=10.10.10.1/30 interface=ether3
add address=30.30.30.2/30 interface=ether4
add address=192.168.1.2/24 interface=ether5
/ip dhcp-server network
add address=192.168.1.0/24 gateway=192.168.1.2
/ip route
add distance=1 dst-address=192.168.2.0/24 gateway=10.10.10.2
add distance=1 dst-address=192.168.3.0/24 gateway=30.30.30.1
/system identity
set name=R01_Moscow
```

### Настройка ПК
На каждом ПК был настроен DHCP-клиент для получения IP-адреса от соответствующего маршрутизатора, а также удалён дефолтный маршрут через сеть управления, чтобы трафик корректно шёл через рабочие интерфейсы локальной сети. Настройки позволили ПК взаимодействовать с другими устройствами в сети.

Пример настройки PC:
```bash
#!/bin/sh
udhcpc -i eth2
ip route del default via 192.168.50.1 dev eth0
```

### Пример работы

После настройки маршрутов и запуска контейнеров была проведена проверка. Результаты выводятся в командной строке маршрутизаторов и ПК.

Команда `ip route print` на маршрутизаторе `R01` показала корректную таблицу маршрутизации:
<img width="1698" height="1046" alt="2025-10-06_17-05-39" src="https://github.com/user-attachments/assets/4a24bd57-9ee8-411f-855a-98a00942c307" />

Аналогичным образом на маршрутизаторе `R02` была показана таблица маршрутизации: 

<img width="1158" height="411" alt="2025-10-06_17-15-11" src="https://github.com/user-attachments/assets/c5f2c7df-b9ec-4b4f-b8c8-d485124eb6df" />

Проверка командой ping с `PC1` до `PC2` и `PC3` показала успешное прохождение пакетов:

<img width="1474" height="1316" alt="2025-10-06_17-17-43" src="https://github.com/user-attachments/assets/c245631e-08b3-4a85-9b78-67a677d410a3" />


## Заключение

В ходе выполнения лабораторной работы была создана и настроена сеть из трёх геораспределенных офисов с использованием маршрутизаторов и клиентских ПК. Были настроены DHCP-сервера, статическая маршрутизация, и проверена корректность работы сети с помощью пингов и трассировки. Все поставленные задачи выполнены, и сеть работает корректно, обеспечивая связь между офисами компании и клиентскими устройствами.
