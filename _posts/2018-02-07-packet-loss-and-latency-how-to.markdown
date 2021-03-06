---
title: Потери пакетов и высокие задержки - как с ними бороться
---

Как правило основной проблемой при запуске бывает неоптимально работающее по умолчанию оборудование (процессор, сетевые карты). Наша задача - поймать все пакеты, не упустив ни одного и обработать, при этом желательно в реальном времени.

Для лучшего понимания незнакомых терминов (IRQ, Softirq, NUMA) - [краткий экскурс в сетевой стек Linux](https://strizhechenko.github.io/2018/01/27/how-network-stack-work-in-7-paragraphs.html).
# Как увидеть информацию о потерях

Для начала надо узнать, что проблема есть. Для этого будем использовать набор утилит netutils-linux и стандартные инструменты iproute. В Carbon Reductor 8 всё поставляется внутри контейнера `/app/reductor`, в других случаях можно установить их следующим образом:

``` shell
yum -y install python-pip
pip install netutils-linux
```

## Информация о конкретной сетевой карте

``` shell
ip -s -s link show eth2
6: eth2: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:e0:ed:33:65:b2 brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast
    66192704908009 358590458063 0       0       0       79155
    RX errors: length   crc     frame   fifo    missed
               0        0       0       0       111495719
    TX: bytes  packets  errors  dropped carrier collsns
    0          0        0       0       0       0
    TX errors: aborted  fifo   window heartbeat
               0        0       0       0
```

Смотреть нужно на все виды RX Errors.

Некоторые сетевые карты предоставляют более подробную информацию о характере потерь:

``` shell
ethtool -S eth2 | egrep rx_ | grep -v ': 0$' | egrep -v 'packets:|bytes:'
     rx_pkts_nic: 365565232680
     rx_bytes_nic: 69973509621284
     rx_missed_errors: 111495719
     rx_long_length_errors: 1077
     rx_csum_offload_errors: 169255
     rx_no_dma_resources: 383638
```

Потери могут быть не только на сетевых картах сервера. Они могут быть и на порту сетевого оборудования, отправляющего зеркало трафика. О том, как это посмотреть можно узнать из документации производителя сетевого оборудования.

## Нагрузка на процессор и что она значит

Выполните команду

```
top -cd1
```

и нажмите клавишу "1", чтобы увидеть информацию о каждом конкретном процессоре.

Вывод имеет следующий формат:

```
Tasks: 143 total,   1 running, 142 sleeping,   0 stopped,   0 zombie
Cpu0  :  0.0%us,  0.0%sy,  0.0%ni, 88.0%id,  0.0%wa,  0.0%hi, 12.0%si,  0.0%st
Cpu1  :  0.0%us,  0.0%sy,  0.0%ni, 88.8%id,  0.0%wa,  0.0%hi, 11.2%si,  0.0%st
Cpu2  :  0.0%us,  1.0%sy,  0.0%ni, 85.0%id,  0.0%wa,  0.0%hi, 14.0%si,  0.0%st
Cpu3  :  0.0%us,  0.0%sy,  0.0%ni, 87.8%id,  0.0%wa,  0.0%hi, 12.2%si,  0.0%st
```
Нас в основном интересуют цифры "%si".

Нагрузка должна быть распределена равномерно, если Cpu0 трудится, а 1..n находятся на нуле - это нерациональное использование имеющихся ресурсов.

- 0% на каждом ядре - а сервер точно работает? Если да - он очень даже хорош.
- 1-3% - всё хорошо настроено, можно даже увеличивать канал и не беспокоиться об апгрейде железа.
- 6-10% - можно не беспокоиться, но перед увеличением канала нужно позаботиться об увеличении мощностей сервера.
- 11-15% - нужно задуматься о покупке более хорошего оборудования или оптимизации настроек имеющегося.
- 20-100% - скорее всего будут потери пакетов. Если ситуация сохраняется после того, как вы прошлись по всем последующим пунктам этой статьи (и воспользовались ими) - свяжитесь с технической поддержкой.

## Информация о сетевом стеке

Вы можете посмотреть подробную информацию о том, как сетевой стек работает в текущий момент.

```
chroot /app/reductor network-top
```

Он подсвечивает некоторые значения жёлтым (высокое значение) и красным (чрезвычайно высокое значение или ошибка). Это эвристика и не подстраивается под конкретный сервер.

При большом количестве ядер или сетевых карт информация может не влезать на экран, в таком случае можно уменьшить масштаб терминала или указать опцию `--devices=eth1,eth2,eth3`:

Вывод выглядит так:

``` shell
[root@reductor support]# network-top -n 1 --no-clear --no-color
# /proc/interrupts

   CPU0    CPU1    CPU2    CPU3    CPU4    CPU5    CPU6    CPU7    CPU8    CPU9   CPU10   CPU11   CPU12   CPU13   CPU14   CPU15

      0       0       0       0       0       0       0     733       0       0       0       0       0       0       0       0   eth0-TxRx-7
  39405       0       0       0       0       0       0       0       0       0       0       0       0       0       0       0   eth2-TxRx-0
      0   39768       0       0       0       0       0       0       0       0       0       0       0       0       0       0   eth2-TxRx-1
      0       0   38518       0       0       0       0       0       0       0       0       0       0       0       0       0   eth2-TxRx-2
      0       0       0   41302       0       0       0       0       0       0       0       0       0       0       0       0   eth2-TxRx-3
      0       0       0       0   39061       0       0       0       0       0       0       0       0       0       0       0   eth2-TxRx-4
      0       0       0       0       0   42078       0       0       0       0       0       0       0       0       0       0   eth2-TxRx-5
      0       0       0       0       0       0   39168       0       0       0       0       0       0       0       0       0   eth2-TxRx-6
      0       0       0       0       0       0       0   39294       0       0       0       0       0       0       0       0   eth2-TxRx-7
      0       0       0       0       0       0       0       0   37167       0       0       0       0       0       0       0   eth2-TxRx-8
      0       0       0       0       0       0       0       0       0   37460       0       0       0       0       0       0   eth2-TxRx-9
      0       0       0       0       0       0       0       0       0       0   40164       0       0       0       0       0   eth2-TxRx-10
      0       0       0       0       0       0       0       0       0       0       0   39613       0       0       0       0   eth2-TxRx-11
      0       0       0       0       0       0       0       0       0       0       0       0   40362       0       0       0   eth2-TxRx-12
      0       0       0       0       0       0       0       0       0       0       0       0       0   41184       0       0   eth2-TxRx-13
      0       0       0       0       0       0       0       0       0       0       0       0       0       0   42319       0   eth2-TxRx-14
      0       0       0       0       0       0       0       0       0       0       0       0       0       0       0   38430   eth2-TxRx-15
    111      61      31      70      23      86      45      77     166       0      12      15      21       9       0     112   interrupts

# Load per cpu:

  CPU     Interrupts   NET RX   NET TX    total   dropped   time_squeeze   cpu_collision   received_rps

  CPU0         39545    39717        0   127558         0              0               0              0
  CPU1         39845    40164        0   130904         0              0               0              0
  CPU2         38594    38925        0   133087         0              0               0              0
  CPU3         41411    41702        0   155217         0              0               0              0
  CPU4         39118    39388        0   126653         0              0               0              0
  CPU5         42188    42443        0   141257         0              0               0              0
  CPU6         39230    39527        0   137838         0              0               0              0
  CPU7         40118    40425        0   117886         0              0               0              0
  CPU8         37335    37430        0   118963         0              0               0              0
  CPU9         37464    37535        0   144810         0              0               0              0
  CPU10        40194    40463        0   135276         0              0               0              0
  CPU11        39630    39953        0   136303         0              0               0              0
  CPU12        40387    40675        0   135858         0              0               0              0
  CPU13        41195    41526        0   134899         0              0               0              0
  CPU14        42321    42705        0   156010         0              0               0              0
  CPU15        38593    38695        0   113177         0              0               0              0

# Network devices

  Device      rx-packets   rx-mbits   rx-errors   dropped   missed   fifo   length   overrun   crc   frame   tx-packets   tx-mbits   tx-errors

  lo                   0          0           0         0        0      0        0         0     0       0            0          0           0
  eth0                21          0           0         0        0      0        0         0     0       0         1155          1           0
  eth1                 0          0           0         0        0      0        0         0     0       0            0          0           0
  eth2           1082133       1462           0         0        0      0        0         0     0       0            0          0           0
  eth3                 0          0           0         0        0      0        0         0     0       0            0          0           0
```

Данные собираются из различных источников (`/proc/interrupts`, `/proc/softirqs`, `/proc/net/softnet_stat` и др) и представленны в удобочитаемом виде. По умолчанию отображается то, насколько данные изменились по сравнению с предыдущими значениями. Есть режим `--no-delta-mode`, отображающий абсолютные значения.

- `/proc/interrupts` отображает то, как очереди сетевой карты распределены между ядрами. Наиболее оптимальный способ - 1 очередь на 1 ядро, можно "лесенкой". Бывает так, что в одну очередь приходит больше пакетов, чем в остальные - обычно это связано с тем, что трафик инкапсулирован (QinQ, PPP).
- Load per CPU - отображает чем занимается каждое ядро. Распределение обработки трафика не обязательно достигается за счёт очередей сетевой карты (RSS) - в некоторых случаях может использоваться технология RPS (программный аналог RSS).
    - `Interrupts` - реальные прерывания, обычно это копирование пакетов из сетевой карты в оперативной памяти.
    - `NET_RX` - отложенные прерывания по обработке входящего трафика: собственно обработка пакетов файрволом
    - `Total` - число обрабатываемых пакетов
    - `dropped` - потери пакетов
    - `time_squeeze` - задержки пакетов - пакет не успел обработаться за отведённое ему время и его решили обработать позже. Не обязательно приводит к потерям, но плохой симптом.
- Network devices - статистика по сетевым картам
    - `rx-errors` - общее число ошибок, обычно суммирует остальные. В какой счётчик попадает потерявшийся пакет зависит от драйвера сетевой карты.
    - `dropped`, `fifo`, `overrun` - как правило, пакеты, не успевшие обработаться сетевым стеком
    - `missed` - как правило, пакеты, не успевшие попасть в сетевой стек
    - `length` - как правило слишком большие пакеты, которые не влезают в MTU на сетевой карте. Лечится его увеличением.
    - `crc` - прилетают битые пакеты. Часто бывает следствием высокой нагрузки на коммутатор.

# Какие вообще бывают настройки

## Процессоры

Частые проблемы в настройках процессора.

### "Плавающая" частота

```
grep '' /sys/devices/system/cpu/cpu0/cpufreq/scaling_{min,cur,max}_freq
/sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq:1600000
/sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq:1600000
/sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq:3201000
```

Суть видна - довольно мощный процессор работает в полсилы и даже не собирается напрягаться. Заставить его работать в полную силу можно используя утилиту `maximize-cpu-freq` входящую в netutils-linux. В Carbon Reductor 8 она используется автоматически.

### Режим энергосбережения

Есть ещё одно "но", которое может приводить к проблемам и которое сложно выяснить программным путём - режим энергосбережения в UEFI/BIOS, который у некоторых процессоров Intel Xeon приводил напротив, к повышению нагрузки. Лучше его выключить, выбрав режим "производительность" (для этого потребуется перезагрузить сервер).

### Гипертрединг

В некоторых случаях использование процессора с отключенным гипертредингом оказывалось эффективнее, чем с включенным, несмотря на меньшее количество логических ядер. Отключить его можно при перезагрузке в BIOS/UEFI.

### Низкая частота

Бывают процессоры с большим числом ядер, но очень низкой частотой - 2 - 2.2GHz. Низкая частота приводит к более медленной обработке отдельно взятого пакета, которая не масштабируется с числом процессоров. Иными словами, числом ядер невозможно побороть задержки в обработке пакетов.

Что с этим делать - лучше не покупать такие процессоры вообще (они могут быть весьма дорогими). Если он уже куплен - поискать ему другое применение.

Оптимальная частота для процессоров используемых для сетевых задач - 3GHz+, но чем выше - тем лучше.

## Сетевые карты

### Размер буфера

```
# ethtool -g eth1
Ring parameters for eth1:
Pre-set maximums:
RX:		4096
RX Mini:	0
RX Jumbo:	0
TX:		4096
Current hardware settings:
RX:		4096
RX Mini:	0
RX Jumbo:	0
TX:		256
```

Здесь мы видим выкрученный на максимум rx-буфер. Обычно подобрать оптимальное значение сложно, большой буфер = задержки, маленький буфер = потери. В нашем случая самое оптимальное - максимальное значение, оптимизации имеют смысл только при наличии проблем.

Пример команд для увеличения буфера:

```
ethtool -G eth1 rx 2048
```

RHEL-based дистрибутивы (платформа Carbon, CentOS, Fedora итд) позволяют указывать параметры ethtool в качестве опции в настройках интерфейса (`/etc/sysconfig/network-scripts/ifcfg-eth1`), например строчкой

```
ETHTOOL_OPTS="-G ${DEVICE} rx 2048"
```

Альтернативный вариант с автоматическим определением оптимального значения (утилита из netutils-linux):

```
rx-buffers-increase eth1
```

### Распределение прерываний

#### Реальные прерывания

Многие сетевые карты имеют несколько очередей для входящих пакетов. Каждая очередь висит на ядре/списке ядер. На многих железках из коробки, несмотря на то, что в smp_affinity_list указан список `0-$cpucount` все прерывания находятся на первом ядре процессора. Обойти это можно распределив все прерывания на разные ядра c помощью утилиты rss-ladder.

```
rss-ladder eth1
```

По возможности используйте разные реальные ядра, гипертрединг лучше выключить, с ним легко получить неоптимальные настройки.

##### Многопроцессорные системы

Если в системе больше одного физического процессора, лучше распределять прерывания сетевой карты в рамках ядер её локальной NUMA-ноды. rss-ladder делает это автоматически. Число очередей лучше подстроить под число ядер одного физического процессора, по умолчанию их число часто равно общему числу ядер.

Пример - поставим eth2 8 объединённых очередей.

``` shell
ethtool -L eth2 combined 8
```

Очереди не всегда бывают combined, бывают отдельные tx и rx, зависит от модели и драйвера сетевой карты.

Не все многопроцессорные системы поддерживают NUMA, иногда память является общей для обоих процессоров и для сетевых карт.

##### Пример для Carbon Reductor 8

Мы не используем автоматическую настройку RSS, т.к. в редких ситуациях это приводит к зависанию сетевой карты. Так что настраивать это необходимо вручную:

Создаем сам файл-хук: `/app/reductor/cfg/userinfo/hooks/start.sh`

В него добавляем следующее содержимое:

```
#!/bin/bash

client_post_start_hook() {
    rss-ladder eth0 || true
    rss-ladder eth1 || true
}
```

и делаем его исполнимым: `chmod a+x /app/reductor/cfg/userinfo/hooks/start.sh`.

#### Отложенные прерывания

Существует технология программного распределения обрабатываемых пакетов между ядрами - RPS. Она универсальна и подходит для любых сетевых карт, даже с одной очередью. Пример настройки:

```
autorps eth2
```

В Carbon Reductor 8 данная утилита используется автоматически для сетевых карт с одной очередью.

### Различные значения rx-usecs

Мы рекомендуем использовать стандартные значения сетевой карты до тех пор, пока не возникнут проблемы.

Статья для лучшего понимания, правда больше под маршрутизаторы : http://habrahabr.ru/post/108240/

В кратце - можно за счёт повышения нагрузки на процессор слегка снять нагрузку с сетёвки уменьшая rx-usecs. На большинстве машин использумых в нашем случае оптимальным оказалось значение 1.

```
ethtool -C eth1 rx-usecs 1
```

### Опции несовместимые с FORWARD / bridge

General Receive Offload и Large Receive Offload могут приводить к паникам ядра Linux и их лучше отключать:

```
ethtool -K eth2 gro off
ethtool -K eth2 lro off
```

Либо при компиляции драйвера.

### Замена сетевых карт

Иногда бывает дело просто в железе. Если уверены, что сетевая карта хорошей модели и есть ещё одна такая же - попробуйте использовать её. Возможно она просто бракованная, хоть вероятность и мала.

Иногда дело бывает в драйвере (в случае dlink / realtek сетевых карт). Они, конечно, здорово поддерживаются практически любым дистрибутивом, но для высоких нагрузок не очень подходят.

## Сетевой стек, iptables

### NOTRACK

Эта технология отключает наблюдение за состоянием соединения для пакетов к которым она была применена. Это приводит к значительному снижению нагрузки на процессор в случае, если состояние соединения нас не интересует (захват трафика).

Крупным провайдерам с большим объёмом трафика эта опция практически обязательна, почему - показывает пример:

#### ДО включения:

```
88.4%si
82.3%si
88.2%si
75.9%si
100.0%si
82.9%si
88.5%si
84.7%si
90.3%si
94.6%si
82.0%si
67.9%si
77.5%si
100.0%si
82.9%si
100.0%si
82.0%si
100.0%si
85.0%si
93.8%si
90.3%si
91.2%si
89.3%si
100.0%si
87.3%si
97.3%si
91.1%si
94.6%si
100.0%si
87.6%si
81.7%si
85.7%si
```

#### ПОСЛЕ включения:

```
1.2%si
2.4%si
0.0%si
0.0%si
0.0%si
0.0%si
0.0%si
0.0%si
0.0%si
1.3%si
1.2%si
0.0%si
1.2%si
0.0%si
0.0%si
0.0%si
0.0%si
0.0%si
0.0%si
0.0%si
1.4%si
0.0%si
1.3%si
0.0%si
0.0%si
0.0%si
0.0%si
0.0%si
0.0%si
1.4%si
1.5%si
0.0%si
```

## Сеть провайдера

### Использование нескольких сетевых карт для приёма зеркала

Вы можете распределить зеркало между несколькими сетевыми картами, указав в настройках создаваемых зеркал равные диапазоны абонентских портов.

### Использование нескольких серверов для приёма зеркала

Вы можете располагать сетевые карты из пункта выше в разных серверах для масштабирования нагрузки.

### Flow-control

Для отправки зеркала рекомендуем отключать данную опцию, она чаще приводит к потерям, чем к сглаживанию пиковых нагрузок.

```
ethtool -A eth2 rx off tx off
```

В Carbon Reductor 8 скоро будет доступно в автоматическом режиме.

### MTU

MTU на порту железки, отправляющей зеркало не должно быть больше, чем MTU интерфейса на Carbon Reductor (в том числе и всех VLAN), принимающего зеркало.

Рекомендуем посмотреть статистику на свитче по распределению размеров пакетов, для D-Link например команда `show packet port 1:1`

и вывод в духе:

```
Port number : 2:11
 Frame Size/Type       Frame Counts                  Frames/sec
 ---------------       ----------------------        -----------
 64                    1470536789                    6330
 65-127                511753536                     12442
 128-255               1145529306                    1433
 256-511               704083758                     1097
 512-1023              842811566                     882
 1024-1518             1348869310                    7004
 1519-1522             2321195608                    1572
 1519-2047             2321195608                    1572
 2048-4095             0                             0
 4096-9216             0                             0
 Unicast RX            0                             0
 Multicast RX          16                            0
 Broadcast RX          0                             0
 Frame Type            Total                         Total/sec
 ---------------       ----------------------        -----------
 RX Bytes              1384                          0
 RX Frames             16                            0
 TX Bytes              20409818277370                15162751
 TX Frames             34114583632                   30760
```

По умолчанию CentOS используется MTU = 1500, лучше выставить его равным максимальному ненулевому значению из статистики.

```
 1519-2047             2321195608                    1572
```

#### Как определить потери пакетов из-за низкого MTU?

Смотрите на RX: length значение в выводе команды:

```
# ip -s -s link show eth1
3: eth1: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1528 qdisc mq state UP qlen 1000
    link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast
    5956390755888 19345313821 3533855 0       0       817154
    RX errors: length   crc     frame   fifo    missed
               3533855  0       0       0       0
    TX: bytes  packets  errors  dropped carrier collsns
    23100      330      0       0       0       0
    TX errors: aborted  fifo   window heartbeat
               0        0       0       0
```

#### Как избавиться от этих потерь?

**Разово** - выполнить команду:

```
ip link set eth1 mtu 1540
```

**Перманентно** - дописать в настройки сетёвой карты `/etc/sysconfig/network-scripts/ifcfg-eth1`:

```
MTU=1540
```

## Carbon Reductor

### Опции снижающие производительность

- menu > настройки алгоритма фильтрации > логировать срабатывания
- menu > настройки алгоритма фильтрации > сохранять домен в адресе редиректа

Они замедляют процесс обработки пакета и отправки редиректов.
