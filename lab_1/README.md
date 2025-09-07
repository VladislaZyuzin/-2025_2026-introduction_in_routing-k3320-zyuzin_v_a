# Отчет по лабораторной работе №1

## Университет
* **University:** [ITMO University](https://itmo.ru/ru/)
* **Faculty:** [ФПиН](https://fict.itmo.ru)
* **Course:** [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)
* **Year:** 2025/2026
* **Group:** K3320
* **Author:** Zyuzin Vladislav Alexandrovich 
* **Lab:** Lab1
* **Date of create:** 01.09.2025
* **Date of finished:** 06.09.2025

## Задание

Вам необходимо сделать трехуровневую сеть связи классического предприятия, изображенную на рисунке 1, в ContainerLab. Необходимо создать все устройства, указанные на схеме ниже, и соединения между ними, правила работы с СontainerLab можно изучить по ссылке.

> **Подсказка №1.** Не забудьте создать mgmt сеть, чтобы можно было зайти на CHR
> 
> **Подсказка №2.** Для mgmt_ipv4 не выбирайте первый и последний адрес в выделенной сети, ходить на CHR можно используя SSH и Telnet (admin/admin)

* Помимо этого вам необходимо настроить IP адреса на интерфейсах и 2 VLAN-a для `PC1` и `PC2`, номера VLAN-ов вы вольны выбрать самостоятельно.
* Также вам необходимо создать 2 DHCP сервера на центральном роутере в ранее созданных VLAN-ах для раздачи IP адресов в них. `PC1` и `PC2` должны получить по 1 IP адресу из своих подсетей.
* Настроить имена устройств, сменить логины и пароли.

![Схема из задания](https://itmo-ict-faculty.github.io/introduction-in-routing/education/labs2023_2024/lab1/3tiernetwork.png)

## Описание работы

Для выполнения работы была арендована виртуальная машина в Selectel. На неё были установлены `docker`, `make` и `containerlab`, а также склонирован репозиторий `hellt/vrnetlab` (в папку routeros был загружен файл chr-6.47.9.vmdk). C помощью `make docker-image` был собран соответствуший образ.

### Топология 
В файле `lab.yaml` описана топология сети. Она включает маршрутизатор `R1`, три коммутатора (`SW1`, `SW2`, `SW3`), а также два конечных устройства (`PC1` и `PC2`). Каждый узел имеет свой файл конфигурации (в папке `configs`, который загружается при старте контейнера.

```
name: lab1

topology:
  nodes:
    R1:
      kind: vr-mikrotik_ros
      image: vrnetlab/vr-routeros:6.47.9
      mgmt-ipv4: 192.168.50.11
      startup-config: ./configs/r1.rsc
    SW1:
      kind: vr-mikrotik_ros
      image: vrnetlab/vr-routeros:6.47.9
      mgmt-ipv4: 192.168.50.12
      startup-config: ./configs/sw1.rsc
    SW2:
      kind: vr-mikrotik_ros
      image: vrnetlab/vr-routeros:6.47.9
      mgmt-ipv4: 192.168.50.13
      startup-config: ./configs/sw2.rsc
    SW3:
      kind: vr-mikrotik_ros
      image: vrnetlab/vr-routeros:6.47.9
      mgmt-ipv4: 192.168.50.14
      startup-config: ./configs/sw3.rsc
    PC1:
      kind: linux
      image: alpine:latest
      binds:
        - ./configs:/configs
    PC2:
      kind: linux
      image: alpine:latest
      binds:
        - ./configs:/configs

  links:
    - endpoints: ["R1:eth2", "SW1:eth2"]
    - endpoints: ["SW1:eth3", "SW2:eth2"]
    - endpoints: ["SW1:eth4", "SW3:eth2"]
    - endpoints: ["SW2:eth3", "PC1:eth2"]
    - endpoints: ["SW3:eth3", "PC2:eth2"]

mgmt:
  network: static
  ipv4-subnet: 192.168.50.0/24
```

Ниже можно ознакомиться с графическим представлением этой схемы (а также с разделением VLAN'ов):

![Топология](images/lab1-topology.svg)

### Настройка маршрутизатора R1
На маршрутизаторе `R1` настроены два VLAN-а — VLAN 10 и VLAN 20. Каждый из этих VLAN используется для разделения трафика между двумя сегментами сети, к которым подключены `PC1` и `PC2`. Для каждого VLAN настроен DHCP-сервер, который автоматически раздаёт IP-адреса устройствам в соответствующих VLAN. Дополнительно был создан новый пользователь с административными правами и изменено имя устройства.

Команды для настройки `R1`:
```
/interface vlan
add name=vlan10 vlan-id=10 interface=ether3
add name=vlan20 vlan-id=20 interface=ether3
/ip address
add address=10.10.10.2/24 interface=vlan10
add address=10.10.20.2/24 interface=vlan20
/ip pool
add name=dhcp_pool_vlan10 ranges=10.10.10.100-10.10.10.200
add name=dhcp_pool_vlan20 ranges=10.10.20.100-10.10.20.200
/ip dhcp-server
add name=dhcp_vlan10 interface=vlan10 address-pool=dhcp_pool_vlan10 disabled=no
add name=dhcp_vlan20 interface=vlan20 address-pool=dhcp_pool_vlan20 disabled=no
/ip dhcp-server network
add address=10.10.10.0/24 gateway=10.10.10.2
add address=10.10.20.0/24 gateway=10.10.20.2
/user add name=newadmin password=newadmin group=full
/system identity set name=R1-Router
```

### Настройка коммутатора SW1
На коммутаторе `SW1` созданы мосты для VLAN 10 и VLAN 20. Это необходимо для того, чтобы трафик в каждом VLAN мог передаваться между различными интерфейсами. Каждый интерфейс, подключенный к определённому VLAN, добавляется в соответствующий мост.

Команды для настройки `SW1`:
```
/interface vlan
add name=vlan10_e3 vlan-id=10 interface=ether3
add name=vlan20_e3 vlan-id=20 interface=ether3
add name=vlan10_e4 vlan-id=10 interface=ether4
add name=vlan20_e5 vlan-id=20 interface=ether5
/interface bridge
add name=br_v10
add name=br_v20
/interface bridge port
add interface=vlan10_e3 bridge=br_v10
add interface=vlan10_e4 bridge=br_v10
add interface=vlan20_e3 bridge=br_v20
add interface=vlan20_e5 bridge=br_v20
/ip dhcp-client
add disabled=no interface=br_v20
add disabled=no interface=br_v10
/user add name=newadmin password=newadmin group=full
/system identity set name=SW1-Switch
```

### Настройка коммутаторов SW2 и SW3
Настройки для `SW2` и `SW3` практически одинаковы, с созданием моста и соответствующих VLAN 10 и 20.

Пример настройки для `SW2`:

```
/interface vlan
add name=vlan10_e3 vlan-id=10 interface=ether3
add name=vlan10_e4 vlan-id=10 interface=ether4
/interface bridge
add name=br_v10
/interface bridge port
add interface=vlan10_e3 bridge=br_v10
add interface=vlan10_e4 bridge=br_v10
/ip dhcp-client
add disabled=no interface=br_v10
/user add name=newadmin password=newadmin group=full
/system identity set name=SW2-Switch
```

### Настройка ПК
Конечные устройства `PC1` и `PC`2 настроены на автоматическое получение IP-адресов через DHCP. Каждое из устройств подключено к соответствующему VLAN: `PC1` в VLAN 10, `PC2` в VLAN 20.

Пример настройки `PC1`:

```
#!/bin/sh
ip link add link eth2 name vlan10 type vlan id 10
ip addr add 10.10.10.10/24 dev vlan10
ip link set vlan10 up
udhcpc -i vlan10
```

### Пример работы

После настройки всех устройств, были выполнены тесты на подключение и маршрутизацию. В частности, `PC1` и `PC2` успешно получили IP-адреса через DHCP, что подтверждается скриншотом ниже:

![Динамическое получение ip адреса для PC1](images/lab1-dhcp-pc1.png)

Также был выполнен тест с помощью команды `ping`, подтверждающий успешное взаимодействие между устройствами в сети:

![Ping PC2 и R1 от PC1](images/lab1-ping-pc1.png)

## Заключение
В результате выполнения лабораторной работы была успешно настроена трехуровневая сеть с VLAN-ами и DHCP-серверами. Все устройства функционируют корректно, и конечные устройства получают IP-адреса согласно настройкам DHCP.
