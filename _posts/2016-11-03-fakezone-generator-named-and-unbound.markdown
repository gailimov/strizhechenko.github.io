---
layout: post
title: Сравнение bind и unbound в целях фильтрации трафика по DNS
date: '2016-11-03 15:48:00'
---

# О чём вообще статья.

Есть такой репозиторий: <https://github.com/carbonsoft/named_fakezone_generator>

Его основная задача - блокировать часть запрещённых сайтов по доменному имени на стороне DNS-сервера провайдера. Оставим моральный аспект этого репозитория за рамками этой статьи, нас интересуют только цифры.

Учитывая то, что реестр запрещённой информации в РФ судя по всему будет только расти, необходимо знать, сколько ресурсов нужно для того, чтобы обеспечивать фильтрацию, достаточную, для того чтобы:

1. Не потерять лицензию / закрыться из-за штрафов.
2. Не потерять абонентов из-за ухудшения качества сервиса.

Общие данные: для каждого добавленного для фильтрации домена необходимо отвечать одним IPv4 адресом.

# Поддержка доменов с подчёркиванием

bind9 от таких доменов, оказавшихся в конфиге валится и не стартует, ругаясь. Одно из решений - заменять на дефис.

unbound чувствует себя прекрасно, на запросы об этих доменах отвечает.

# Как генерировались домены

На коленке был накидан скрипт на python2, красоту наводить не стали, ибо скрипт одноразовый.

```python
""" domain generator """
from re import sub
from sys import argv
from random import shuffle
from string import ascii_lowercase

zones = [
    'com', 'net', 'ru', 'org', 'uk', 'en', 'ss', 'cn',
]

x = list(ascii_lowercase) + ['1', '2', '3', '4', '.', '.', '.']

for i in xrange(int(argv[1])):
    shuffle(x)
    print sub(r'^\.|\.$', '', "".join(x).replace('..', '.').replace('..', '.')) + '.' + zones[i % len(zones)]
```

На выходе даёт что-то вроде:

```
3zxkljdpytqwev4u1.mgconis2hbaf.r.net
3.gxchap1zu4wt2rvejyqln.bodfms.ik.ru
p2beiw4r.uzydlkhnq.jcf3.xto1gsvma.org
r1ve.os2p.ilzdt4kj.baqycuwfmng3hx.uk
snemj1d.of2huzqkxbcilvag.43ywrpt.en
cbek1pmvlzhowg.st.iy2nx.jq4da3fru.ss
rgct.jqop2l1n3zums.4bayxvkw.eihdf.cn
xecabuy124sdgpqith.omw3.rjlznkvf.com
4zr1mlx3howu.nciajkvb.gq.syptf2ed.net
nh.iq1gxfclt.rpsd4z3j.aevu2bmwoky.ru
ylr.fcvjzumk42q.ontx3g.easwbdp1hi.org
bw4vzrkaenfj.lpsho.ug21.mctyxiq3d.uk
n1gvqf4cwmjkrzphy.32ouxdtaei.bsl.en
qwv.tpgknfjhi.ey.ozrmbx1c3u2ads4l.ss
```

Для тестов вполне подходящие данные, хоть таких доменов скорее всего на самом деле и не существует.

# Потребление памяти

Формат записи: 1 число - VSZ, 2 - RSS из вывода ps aux. Наблюдения проводились за сервером, находящимся в состоянии "покоя" - к нему обращалась несколько раз тестовая машина только для того, чтобы проверить, что он вообще работает и живой. Оба сервера были развёрнуты в OpenVZ-контейнерах, обоим было выделено 4гб оперативной памяти.

Число доменов | Unbound VSZ | Unbound RSS | Bind9 VSZ | Bind9 RSS
:-----------: | :---------: | :---------: | :-------: | :-------:
    8500      |    210M     |     49M     |   357M    |   101M
    25000     |    346M     |    157M     |   970M    |   306M
    50000     |    556M     |    315M     |   1299M   |   646M
   100000     |    979M     |    823M     | OOM | OOM
   200000     |    1824M    |    1284M    | OOM | OOM
   400000     |    3515M    |    2655M    | OOM | OOM

# Что стоило/стоит ещё померять

- Деградация в скорости ответов
- Замедление рестарта
- PowerDNS и PowerDNS-Recursor
- DNS-спуфер в Carbon Reductor
- Bind10
- NSD сам по себе
- Связка Bind + NSD
- Связка Unbound + NSD

# Выводы

После 50000 доменов наступает существенная деградация в скорости рестарта у bind9.

На 400000 доменов скорость полного рестарта (с downtime!) у unbound замедляется до ≈14 секунд, что тоже грустно:

```shell
# time /etc/init.d/unbound stop; time /etc/init.d/unbound start
Stopping unbound:                                          [  OK  ]

real	0m1.116s
user	0m0.007s
sys	0m0.003s
Starting unbound: Nov 04 09:23:14 unbound[30785:0] warning: increased limit(open files) from 1024 to 8290
[  OK  ]

real	0m14.791s
user	0m8.548s
sys	0m3.865s
```

К счастью у unbound есть утилита unbound-control, позволяющая менять конфигурацию сервера на ходу, без downtime, а в named fakezone generator - поддержка заливки только изменений с её помощью.

На 100000 без дополнительного тюнинга bind в openvz начал валиться с out of memory error прямо при старте, съев около 1гб памяти.

Для одного пользователя, решившего опросить все добавленные домены с localhost потребление памяти ни unbound'ом ни bind'ом не менялось.

# Размышления

Вообще идея использовать для фильтрации DNS-сервера может показаться довольно странной. Однако DNSSec никаким спуфингом сбоку не побороть, разве что установкой фильтрации в разрыв (что на мой взгляд куда хуже, чем интеграция с DNS-сервером).

## Unbound

У unbound есть одна интересная опция: - ```domain-insecure: "example.com"```. Она позволяет обрабатывать запросы от unbound модулем DNS-спуфинга Carbon Reductor.

Но мне не удалось найти способа применять её иным способом, кроме перечитывания конфига (unbound-control этого не умеет). Это позволяет генерировать конфиг значительно меньшего размера. Правда на удивление это мало влияет на потребляемую память (по крайней мере на 8500 доменов разницы не было - 11мб RSS).
