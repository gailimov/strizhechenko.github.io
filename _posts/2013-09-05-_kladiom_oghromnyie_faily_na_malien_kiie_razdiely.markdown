---
layout: post
title: '"Кладём" огромные файлы на маленькие разделы'
date: '2013-09-05 09:14:00'
---

Бывают ситуации, когда на небольшой раздел диска нужно положить файл, значительно превышающий объём свободного места, в другое место вот положить никак нельзя. При этом есть несколько разделов с бОльшим объёмом, где собственно этот файл уже лежит.

Хардлинки вряд-ли подойдут, из-за "invalid cross-device link", но возможно я просто не умею ими пользоваться. Выйти из такого положения достаточно просто:

К примеру папка /var/backup/db/ лежит как раз на таком разделе, где мало места (1.3гб). Файл daily.gbk весит 1.4гб. Вот что мы делаем:

[root@www-example-com root]# &gt; /var/backup/db/daily.gbk
[root@www-example-com root]# mount --bind daily.gbk /var/backup/db/daily.gbk

Профит - имеем незанимающий места на разделе файл в полтора гига :)