---
layout: post
title: "Как запихать содержимое файла в char[]"
date: '2013-09-02 16:18:00'
tags:
- bash2
- c
- sed
- char
- hex
- hexdump
- printf
---

hordecore@oleg:~$ hexdump -v -e '12/1 "0x%02X, " "\n"' file.txt | sed 's/0x  /0x00/g'
0x79, 0x6F, 0x75, 0x20, 0x61, 0x72, 0x65, 0x20, 0x64, 0x69, 0x72, 0x74,
0x79, 0x20, 0x61, 0x73, 0x73, 0x68, 0x6F, 0x6C, 0x65, 0x73, 0x0A, 0x  ,

вывод выделяем, копируем, считаем любым способом число байтов, вставляем в свой сишничек и окружаем в конструкцию:

char data[24] = {
0x79, 0x6F, 0x75, 0x20, 0x61, 0x72, 0x65, 0x20, 0x64, 0x69, 0x72, 0x74,
0x79, 0x20, 0x61, 0x73, 0x73, 0x68, 0x6F, 0x6C, 0x65, 0x73, 0x0A, 0x  ,
};

профит :)

На всякий случай проверяйте что вставили всё верно, сделайте printf / printk:

printf("%s\n", data);
Результат примера:

hordecore@oleg:~$ gcc test.c 
hordecore@oleg:~$ ./a.out 
you are dirty assholes