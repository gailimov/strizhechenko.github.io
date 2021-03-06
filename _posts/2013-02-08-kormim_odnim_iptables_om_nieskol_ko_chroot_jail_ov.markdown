---
layout: post
title: "Кормим одним iptables'ом несколько chroot jail'ов"
date: '2013-02-08 07:49:00'
---

Сегодня наконец решил проблему с файрволом. Вкратце - платформа Carbon - это только базовая система, позволяющая стартовать на ней сервисы, являющиеся чрутами с некоторыми дополнениями.
А проблема в том, что сервисов много, а файрвол один, и при стопе одного приложения, не должны удаляться правила другого.
Решение достаточно простое. В оригинальных цепочках создавать цепочки с префиксом конкретного сервиса при старте, добавлять в них правило-ссылку. При стопе же делать:

        for table in filter nat mangle; do
                while read -t 3 rule; do
                        if [[ "$rule" = *"-j nas_"* ]]; then
                                iptables -t $table ${rule//-A/-D}
                        fi
                done <<< "$(iptables-save -t $table)"
        done
        for table in filter nat mangle; do
                while read -t 3 tmp chain tmp; do
                        [ -z "$table" -o -z "$chain" ] && continue
                        iptables -t $table -F $chain
                        iptables -t $table -X $chain
                done <<< "$(iptables -t $table -nvL | fgrep "Chain nas_")"
        done

А потом проходиться по списку цепочек с этим префиксом, во славу грепа, сперва -Fлуша, а затем -Херя их в тартарары.
Обязательно ссылки удалять перед самими цепочками, так как иначе цепочки не будут удалены.
Такой вот нехитрый способ позволит не трогать чужие правила, и добавленные вручную в частности.
Единственное, о чем я пока не задумывался, это о том, что делать с ipset. В принципе использует его пока только carbon as (из наших продуктов, конечно), так что можно хоть все полностью вычищать, что я до тех пор пока не понадобится, и делаю.
Кстати, этот способ сильно упростил процедуру портмаппинга, достаточно одной цепочки, которая при рестарте флушится, и заполняется из конфига заново, для нее быстро была допилена функция персонального флуша-перезаполнения.
В принципе добить удаление/добавление записей в виджеты table - и я готов лично обзванивать клиентов и предлагать им поюзать насы, потому что мне не будет стыдно за него, ибо то что он делает - он делает хорошо.