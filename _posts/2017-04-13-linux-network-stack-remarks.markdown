---
title: Как работает Linux Network Stack, как его тюнить и мониторить
---

Источник: [Блог packagecloud.io](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/)

## Итак, путь пакетика из кабеля в приложение приблизительно такой:

1. Прилетел сигнал в сетёвку
2. Сетёвка через DMA пишет свою память с пакетом в RAM
3. После чего вызывает прерывание (IRQ, то, что видно в /proc/interrupts), которое оповещает ядро о том что пакет пришёл
4. Вызывается функция (IRQ handler), которую зарегистрировал драйвер сетёвки при иниициализации.
5. IRQ завершается, в контексте этого IRQ делаются совсем уж копейки, чот вроде отложенной пометки что пора пакеты обрабатывать.
6. Дальше в дело вступает NAPI
7. `napi_schedule()` добавляет NAPI poll structure в `poll_list` для текущего CPU и где-то выставляет бит о том что ПОРА БЫ И SOFTIRQ БАХНУТЬ! Насколько я понимаю, текущий CPU это тот, кем обработалась очередь в которую пришёл пакет.
8. Так как бит выставлен, ksoftirqd работающий на данном CPU это видит.
9. Гоняемая в цикле `run_ksoftirqd()` функция выполняется.
10. `__do_softirq()` вызывает функцию, прибитую к NET_RX прерыванию, то есть `net_rx_action()`. Напомню, исполняет её УЖЕ НЕ ДРАЙВЕР, а тред ksoftirqd в ядре.
11. Теперь мы в контексте softirq. Это и есть тот самый %si, /proc/softirqs.
12. `net_rx_action` в цикле проверяет NAPI `poll_list` на наличие NAPI структур.
13. Проверяет, что `budget` и elapsed time не израсходованы и softirq не выжрало весь CPU.
14. Вызывается poll-функция зареганная драйвером при инициализации, для igb это `igb_poll()`.
15. Задача poll-функции - выжирать пакеты из кольцевых буферов в оперативной памяти (ethtool -g eth1, вот эти самые). Тут у меня всё ещё есть один пробел в знаниях, я так и не понимаю, размер ring буфера измеряется в числе дескрипторов, но я не помню, соотносятся эти дескрипторы 1 к 1 с пакетами или нет. + ring буферов столько, сколько очередей у сетёвки.
16. Дальше происходит `napi_gro_receive()` GRO (Generic Receive Offloading), если включено, это шняга очень помогающая при чистом INPUT и опасная (+ вроде бы абсолютно бесполезная) при FORWARD. Вообще тоже стоит про неё повнимательнее почитать. При включении пакеты складываются в GRO list.
17. Если GRO отключено пакеты попадают непосредственно в `net_receive_skb()`
18. `net_receive_skb()` это место, где вызывается или не вызывается **RPS**, в зависимости от того, включен он или нет. Если включен:
    1. Пакет пихается в бэклог с помощью `enqueue_to_backlog()`. Насчёт этого backlog у меня есть подозрение что `/proc/sys/net/core/netdev_max_backlog` - это он и есть. Насколько я понимаю, его можно рассматривать как дополнительный промежуточный RX-буфер, который существует для каждого CPU, даже если очередь у драйвера всего одна.
    2. Пакеты раскидываются между доступными CPU (указанными в rps_cpus очереди) для дальнейшей обработки.
    3. NAPI-структура добавляется в `poll_list` CPU, на который назначился пакет, в очередь ставится IPI (Inter-processor Interrupt), который вызовет softirq.
    4. В процессе работы ksoftirqd на назначенном CPU затем происходит то же что и выше, но poll-функция уже не `igb_poll()`, а `process_backlog()` которая разгребает input queue данного CPU.
19. Вне зависимости от RPS дальше пакет попадает в `__net_receive_skb_core` (или `__netif_receive_core()` или `__netif_receive_skb_core()`), дальше перепрыгиваний пакета с ядра на ядро^Wраспределения пакетов между ядрами уже не будет.
20. `__net_receive_skb_core()` уже имеет дело с skb, который мы в последствии и анализируем. Задача этой функции - доставлять пакеты к taps (PCAP является одним из них).
21. После taps'ов пакет попадает выше по стеку на L3, например для IPv4 это будет `ip_rcv()`
22. Дальше происходит вся фигня связанная с netfilter, iptables и роутингом
23. Дальше пакет попадает в свой UDP/TCP-стек, например в `udp_rcv()`
24. `udp_rcv()` кладёт пакет в очередь на отправку в сокет пользовательского пространства с помощью функции `udp_queue_rcv_skb()`.
25. Но перед тем как пакет попадёт в очередь к нему применяются BPF (Berkley Packet Filters) и он может до сокета и не дойти.

## Какие можно сделать выводы отсюда?

Есть два способа распределить нагрузку по обработке пакетов между ядрами процессора:

1. RSS - назначить `smp_affinity` для каждой очереди сетёвки.
2. RPS - назначить `rps_cpus` для каждой очереди сетёвки.

Эти опции можно комбинировать. Эффект при этом мне пока предсказать сложно - дополнительная очередь:

С одной стороны снижает вероятность потери пакета при пиковой нагрузке ценой общего увеличения времени обработки

Для сетевых карт с несколькими очередями и равномерным распределением пакетов перекидывание пакета с ядра на ядро - возможно дороговатая задача и бессмысленная задача, которая будет кушать только budget и тратить время + удваивать число softirq на пакет + реже/позже будут срабатывать попадания данных пакета в кэш (опять же зависит от кучи факторов, без замеров о чём-то говорить - зло).

Имхо в моём случае самая жирная задача из всего - копание в данных самого пакета, но пренебрегать возможностями тюнинга низлежащих уровней - грех.

## Как мониторить и тюнить сетевой стек (не особо в него вникая)

Мониторинг стоит сразу же разделить на краткосрочный (посмотреть как чувствует себя система прямо сейчас) и долгосрочный (с алертами, вот это всё).

Заниматься тюнингом не имея под руками инструментов хотя бы для краткосрочного мониторинга - зло и трэш. Поэтому я эти инструменты запилил^Wнавелосипедил (для себя в первую очередь)

``` shell
pip install netutils-linux
```

Обкатано на CentOS 6.9 x86_64 + python 2.6, тесты гоняются в python 2.7.

В принципе на [github](https://github.com/strizhechenko/netutils-linux) можно присылать issue и pull-request'ы если что-то не заработает на другой системе, буду рад пофиксить.

В итоге получаем следующие утилиты.

### Мониторинг

#### softnet-stat-top

Можно посмотреть сколько пакетов через какие ядра проходит и насколько успешно обрабатывается.

По сути для хорошей нужно добиваться чтобы прирост total был максимально равномерным между CPU, а счётчики dropped / time_squeeze не росли вообще.

```
$ softnet-stat-top --help
Usage: softnet-stat-top [options]

Options:
  -h, --help            show this help message and exit
  -i INTERVAL, --interval=INTERVAL
                        specifies the delay between screen updates
  -a, --assert-mode     stops screen updates if there is packets drop
  -u, --unit-tests      runs unit-tests instead of read anything from system
  -n, --no-delta-mode   shows actual counters instead evaluating deltas
```

Так можно посмотреть текущее распределение нагрузки

```
# softnet-stat-top
Press CTRL-C to exit.
CPU:  0 total:       78 dropped: 0 time_squeeze: 0 cpu_collision: 0 received_rps: 0
CPU:  1 total:       83 dropped: 0 time_squeeze: 0 cpu_collision: 0 received_rps: 0
```

Добавив параметр `-a` можно залипать до тех пор, пока не потеряется хотя бы один пакет.

Параметр `-n` позволит наблюдать реальные значения счётчиков за всё время работы системы, а не их прирост.

```
# softnet-stat-top -n
Press CTRL-C to exit.
CPU: 0 total:   154980326 dropped: 0 time_squeeze:    4 cpu_collision: 0 received_rps: 0
CPU: 1 total:   158604587 dropped: 0 time_squeeze:    4 cpu_collision: 0 received_rps: 0
```

#### softirq-net-rx-top

Позволяет наблюдать за приростом числа SoftIRQ NET_RX, который читается из /proc/softirqs, попутно наблюдая за uptime.

Эта утилита нужна в большинстве случаев для отладки настройки RPS, потому что с этими долбанными логическими ядрами на многопроцессорных системах в Linux и их масками можно шизануться.

Вот к примеру вывод на машине с двумя ядрами:

```
$ lscpu -e
CPU NODE SOCKET CORE L1d:L1i:L2 ONLINE
0   0    0      0    0:0:0      да
1   0    0      1    1:1:0      да
```

а в /proc/softirqs ядер 4.

```
$ softirq-net-rx-top
0.26, 0.28, 0.25
1	73
2	96
3	0
4	0
```

#### irqtop

Позволяет наблюдать за приростом IRQ, по умолчанию из вида скрываются те прерывания, счётчики которых не растут.

Нужно чтобы оценить правильность работы RSS. Оптимальный вариант - когда Total в верхней строчке максимально равномерен.

```
$ irqtop
Total:	1064	1971
 	CPU0	CPU1
28:	374	277	PCI-MSI-edge	eth0
29:	395	493	PCI-MSI-edge	eth1
LOC:	41	1004	Local	timer	interrupts
CAL:	254	195	Function	call	interrupts
```

#### link-rate

Утилита, показывающая число пакетов и байтов, прилетающих на сетёвку/улетающих с сетёвки в секунду. Данные дёргаются из sysfs.

Человеческого формата пока не имеет, но он вроде особо и не нужен.

```
$ link-rate eth1
rx_bytes:14 336 	rx_packets:47 	tx_bytes:0 	tx_packets:0
rx_bytes:37 641 	rx_packets:145 	tx_bytes:0 	tx_packets:0
rx_bytes:14 852 	rx_packets:63 	tx_bytes:0 	tx_packets:0
rx_bytes:14 416 	rx_packets:53 	tx_bytes:0 	tx_packets:0
rx_bytes:23 648 	rx_packets:56 	tx_bytes:0 	tx_packets:0
```

#### server-info

Тулза для определения насколько железо хорошо для работы с сетью. Пока заточена только под захват трафика, позже возможно будут добавлены профили transmit / forward / receive.

Вывод, кстати, в YAML, сразу и человеко и машинно-читаемо выходит.

Умеет показывать само железо, если хочется посмотреть глазами.

``` yaml
$ server-info show
cpu:
  info:
    Architecture: x86_64
    BogoMIPS: 6118.0600000000004
    Byte Order: Little Endian
    CPU MHz: 3059.0300000000002
    CPU family: 6
    CPU op-mode(s): 32-bit, 64-bit
    CPU(s): 2
    Core(s) per socket: 2
    L1d cache: 32K
    L1i cache: 32K
    L2 cache: 2048K
    Model: 23
    Model name: Pentium(R) Dual-Core  CPU      E6600  @ 3.06GHz
    NUMA node(s): 1
    NUMA node0 CPU(s): 0,1
    On-line CPU(s) list: 0,1
    Socket(s): 1
    Stepping: 10
    Thread(s) per core: 1
    Vendor ID: GenuineIntel
    Virtualization: VT-x
  layout:
    '0': '0'
    '1': '0'
disk:
  sda:
    model: ST1000DM003-1ER1
    size: 1000204886016
    type: HDD
memory:
  MemFree: 304516
  MemTotal: 3204560
  SwapFree: 3341100
  SwapTotal: 3342332
net:
  eth1:
    buffers: null
    conf:
      ip: ''
      vlan: false
    driver:
      driver: r8169
      version: 2.3LK-NAPI
    queues:
      own:
      - eth1
      rx: []
      rxtx: []
      shared: []
      tx: []
      unknown: []
```

и оценивать это железо по шкале от 1 до 10. Когда-нибудь добавлю параметр, позволяющий обобщать оценку вплоть до верхнего уровня.

``` yaml
$ server-info rate
cpu:
  BogoMIPS: 6
  CPU MHz: 6
  CPU(s): 1
  Core(s) per socket: 10
  L3 cache: 1
  Socket(s): 1
  Thread(s) per core: 10
  Vendor ID: 10
disk:
  sda:
    size: 10
    type: 1
memory:
  MemTotal: 2
  SwapTotal: 8
net:
  eth1:
    buffers:
      cur: 1
      max: 1
    driver: 1
    queues: 1
system:
  Hypervisor vendor: 10
  Virtualization type: 10
```

### Тюнинг

#### maximize-cpu-freq

Плавающая частота - не для сетевых девайсов вообще. Если есть проц который может пахать на 3.5GHz - пусть пашет на них, а не уходит в 1.2GHz чтобы сэкономить немного ватт.

Включает для cpu_freq_governour режим performance + прописывает максимальную частоту в минимальную.

#### rss-ladder

Автоматическое распределение прерываний "лесенкой" для multiqueue-сетевых карт.

Учитывает то, что есть NUMA и L3 кэши и в многопроцессорных системах лучше сидеть всеми прерываниями одной сетевой карты на одном физическом проце.

При использовании нескольких сетевых карт в идеале добиться того что для каждой очереди будет одно физическое ядро, ответственное только за неё.

Если ядер не хватает - число очередей можно уменьшить с помощью ethtool, например:

```
ethtool -L eth0 combined 2
```

``` shell
$ rss-ladder eth0 1
- Распределение прерываний eth0 (-TxRx) на сокете 1
  - eth0: irq 24 eth0-TxRx-0 -> 4
  - eth0: irq 25 eth0-TxRx-1 -> 5
  - eth0: irq 26 eth0-TxRx-2 -> 6
  - eth0: irq 27 eth0-TxRx-3 -> 7
```

#### rx-buffers-increase

Увеличивает RX-буфер до довольно хитрым образом вычисляемого размера, если это возможно.

На деле, наверное, стоит поправить и всегда выкручивать в максимум.

```
# rx-buffers-increase eth1
run: ethtool -G eth1 rx 2048
# rx-buffers-increase eth1
eth1's RX ring buffer already has fine size.
```

#### autorps

Утилита для случаев, когда проблемы с бюджетом велики, а улучшить производительность плохого железа надо хоть как-то.

Нужна для сетевых карт с одной очередью, работающих на системе с одним физическим процессором, чтобы раскидать обработку пакетов между всеми ядрами.

Автоматически считает и прописывает в нужные файлы маску процессоров для RPS.

Если всё работает нормально, работает молча, если есть какие-то проблемы - пишет о них (как в моём случае).

```
$ autorps eth1
This system has more then one socket (2).
/usr/bin/autorps in current implementation may cause performance degradation in this case, so we skip it.
```
