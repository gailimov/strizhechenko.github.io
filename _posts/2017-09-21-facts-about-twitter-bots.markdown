---
title: Факты о твиттер-ботах.
---

1. Он почти копипаста, мне пришлось поменять только:
    1. id базы в redis
    2. шаблон фразы
    3. морфологический фильтр слов
2. Он написан на python 2.7, крутится внутри OpenVZ контейнера сильно порезанными ресурсами (256мб оперативки) на моём рабочем сервере.
3. На самом деле я очень беспокоюсь, что этот бот кого-нибудь сильно обидит.
4. Либа для написания твиттер-ботов - мой первый Open Source проект. Первый бот - https://twitter.com/vsevsezaebali, но он меня заебал и я даже хз, пашет, не.
5. После той либы я сделал надстройку над ней, теперь у меня есть враппер, удобный для разворачивания фермы твиттер-ботов.
6. Всего есть 2 вида ботов - reader и writer. Reader пишет всю свою ленту в базу redis, writer'ы из неё читают
Reader только один - мой акк
7. Reader чистит свою базу, находя минимальный использованный всеми writer'ами твит, удаляя его и всё что старее. Ферма живёт автономно.
8. Я думал о том, как монетизировать everyword-ботов, но аудитория не растёт быстро, так что до 10к подписчиков срал ебал этим заниматься.
9. Я стесняюсь написать NLP в резюме, хотя это очень интересная для меня тема. Дело в том, что я в основном занимаюсь сетями и Linux Kernel.
10. Для работы с склонением слов я использую pymorphy2, это пожалуй самое крутое что есть для русского языка.
https://github.com/kmike/pymorphy2
11. Ещё один клёвый проект который упрощает мне жизнь во всех этих ботах - dictator. Знаете кто ещё его использовал? Гитлер!
12. На самом деле идея этого бота не моя, это всё https://twitter.com/hpstrsx!
13. Всё лежит на github:
https://github.com/strizhechenko/twitterbot-utils
https://github.com/strizhechenko/twitterbot_farm
14. Раньше боты крутились в heroku, но с ростом их числа хватать free hours перестало хватать. Потом был digitalocean, а потом жаба задавила
15. Бот https://twitter.com/__coding_tips__ сделан на другом движке и на текущий момент является не совсем ботом, скорее bot-assisted account.
    1. Он написан на golang, имеет бэкенд и полурабочий фронтенд, т.к. фронтенд не дописан, я копипащу твиты в десктопный клиент ручками.
    2. У него есть чёрный список паттернов, потому что люди, блядь, годами пишут одну и ту же залупу в твиттер.
    3. По той же причине сейчас этот бот требует ручной модерации. Если постить абсолютно всё - выйдет так себе: https://twitter.com/coding_tips_sys
16. Интересным отклонением от "стандарта" было написание https://twitter.com/HarryTwibotter, который выискивает в твитах паттерн из двух слов.
17. Вообще все эти боты напоминают мне автоматизированные текстовые рисовач треды с 0chan.
18. Вообще всё началось с того, что я кормил всякое дерьмо цепям маркова, перемешивая с твиттером. Саша Лавкрафт Конь, Феминистки и Погром.
19. Не представляю сколько места и памяти все эти боты жрали, если бы я пытался запихать их в docker. Сейчас: ≈2гб, все базы - 11мб.
20. Контейнеру с этими ботами около года. По моим прикидкам его хватит ещё на 672 года работы без проблем.
21. Использование в качестве источника слов ленты твиттера потенциально делает ботов абсолютно автономными, люди придумывают новые слова.
22. У ботов есть "мониторинг". Influxdb+Grafana+Cron+Bash показывают сколько слов есть в запасе для обработки.
P.S: retention policy рулят.
23. Когда-нибудь я напишу твиттер-бота на чистом C.
24. Возможно даже в kernelspace.
25. Я всё ещё не понимаю как вы всё это читаете, но заняться аналитикой и посмотреть насколько велика текучка фолловеров лень.
26. Я давно скидал всех ботов в https://twitter.com/strizhechenko/lists/list и доволен.
27. Ещё про redis - я охуеваю с того, что ему ничего не делается, несмотря на то, что сервак падал в паники, его ребутили по питанию итд.
28. То же самое касается ext4 внутри ploop у openvz. Не ломаются и всё тут.
29. Как-то раз я пустил в качестве источника слов песенки гражданской обороны по кругу и всех заебал.
30. Я так и не поэкспериментировал с зависимостью скорости прироста читателей и частоты постинга. Поэтому все боты пишут раз в час.
31. После появления в моей жизни https://github.com/strizhechenko/netutils-linux я нихера не делал с твиттер-ботами, потому что это куда круче для портфолио.
32. Боты с использованием цепей Маркова никому не интересны. Они генерят ≈5% забавной херни в лучшем случае.
33. Давно хочу попробовать поиграть уже с RNN и чем-нибудь подобным, но не знаю с какой стороны подступиться.
34. Вообще интерес к ботам у меня начался с World of Warcraft. Я играл на пиратском сервере, юзал speedhack и в два окна гриндил Саурфанга.
    1. 1й персонаж был внутри текстур, 2й в воздухе бил, агр постоянно перекидывался на 1го, так что нужно было обеспечисть много тиков/сек.
    2. Спамить ротацию дестролока руками по полтора часа заебало ОЧЕНЬ быстро. AutoIT в руки и вуаля - стабильные 12к дпс с регеном маны.
35. Это всё на самом деле вытекло из автоматизации и тыканья в xdotool на работе.
36. Я экспериментировал с ботами, которые очень аккуратно пиздят твиты и фолловят случайных людей. Успешно.
    1. И да, именно с целью накрутки-продажи. В принципе если задрочиться - это около 2 лет назад могло бы быть выгодным бизнесом, сейчас хз.
    2. Почему забросил - где-то годик назад меня переторкнуло чтить EULA почти везде. + Качественным AI это не пахло и вообще очень скучно.
37. У твиттера один из самых лучших API из всех социальных сетей, с которыми я работал. + У него чертовски удобная регистрация аккаунтов.
38. Вся прелесть в том, что он не требует никаких callback url чтобы подписать юзера на приложение, вы просто копируете пин-код из браузера.
39. Благодаря этому вы можете начать клепать ботов без вложения бабла на VPS, сидя за натом на локалхосте.
40. Про машинное обучение - класс бота Analyzer в twitterbot-farm так и остался нетронутым.
41. Но идея была примитивной до жути: не/мало полайкали твит в течение 24 часов, помещаем твитообразуюзее слова в глобальный блэклист.
42. Хорошо полайкали твит - рассылаем твитообразующее слово по всем ботам.
43. Где-то в документации твиттера сказано, что они приветствуют ботов, если это полезно людям. Ну или нравится им. Это очень круто.
44. Есть несколько ботов, которых я случайно забросил. Ну или не случайно, показалось черезчур однопетросянообразно:
https://twitter.com/boring_sexism
45. Самый крутой и РЕАЛЬНО ПОЛЕЗНЫЙ бот: лень было натравить его на кого-то кроме лепры, а потом...
https://twitter.com/TweetThiefTrack
46. Помимо самого бота, лепразориум забанил еще и мой основной аккаунт до кучи. Что в целом к лучшему.
47. С помощью только твиттер API вы не сможете удалить все свои твиты. А с помощью сторонних сервисов вы раздадите им кучу разрешений на акк
48. Вы можете подтянуть только последние 3200 твитов юзера с помощью API. Но свои твиты можно скачать архивом где-то в настройках.
49. У меня есть скрипт, идущий по этому архиву и вычищающий всё. Это дело пары дней. https://github.com/strizhechenko/netutils-linux
50. До того, как делать ботов на отдельных аккаунтах, я мучал своих фолловеров автоматической сменой аватарки и ника.
51. Это было что-то в духе (о, мой бот!) https://twitter.com/name4tumblr но из 8 слов. Код/шаурма/кислота, что-то там ещё. 2^8 = 256, хватало почти на год.
52. Этот бот, кстати получил какой-то упоротый словарь слов русского языка и постит всякую главновсибвнерыбу. А еще на 50% написан на bash.
53. Я устал и иду спать, да и факты вроде кончились. Могу завтра с утра добить ответами на вопросы.
54. Периодически попадаются люди, которые думают, что этот бот реальный человек который сутками написывает и написывает. И не спит.
55. Лебедев начал постить однообразную шаблонную херню без смысла раньше.
56. Каждый ~700й твит написан ручками под настроение.
57. Я всё ещё согласен с закрепленным твитом: https://twitter.com/eto_kogda_ebut/status/690111846817370113
58. Изначально я хотел использовать NLTK, но его было тяжело тазить в хероку. Точнее его датасеты.
59. Ради того, чтобы сделать этих ботов ещё пиздатее я просмотрел 2 недели курса по Machine Learning на курсере и забил на это дело.
60. Если за первую неделю бот не набрал сотню читателей, то так в её пределах крутиться и будет.
61. Перед тем, как публиковать бота, нужно нагенерить хотя бы 100-200 твитов.
62. Если сделать это после публикации - первые фолловеры охуеют с такого потока в ленте.
63. Если не сделать этого вообще, то "конверсия" первых читателей будет низкой, потому что ему ещё толком нечем зацепить читателя.
64. Публикация – громко сказано. Это ретвит с основного аккаунта и пары других аккаунтов ботов.
65. 5-6 ретвитов тысячников в первую неделю существования бота решают его дальнейшую судьбу.
66. А может и не решают, потому что это тупо ебучий скрипт, которому срать на все лайки-ретвиты, он возможно ещё вас переживёт.
