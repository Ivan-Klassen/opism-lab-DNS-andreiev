<!--
Бланк звіту до лабораторної роботи «DNS».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Коментарі такого вигляду в GitHub не відображаються; видаляти їх не обов'язково.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота. DNS

| Поле | Значення |
|---|---|
| ПІБ | Андреєв Іван Олегович |
| Група | ІПЗ_2.01 |
| Ідентифікатор (`sNN`) | se-201-01 |
| Дата виконання | 15.11.26 |
| Середовище | Docker на Debian Linux |
| Версія `dig` | DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu |

## Параметри варіанта

<!-- Переписати з картки stand/cards/sNN.md. -->

| Параметр | Значення |
|---|---|
| `$TTL`, с | 60 |
| TTL запису SOA, с | 20 |
| Поле `minimum` SOA, с | 90 |
| Псевдонім (CNAME на `www`) | blog |
| Другий хост і його TTL, с | edge, 50 |
| Пріоритет MX | 25 |
| Токен TXT | 3bc4ea6d |
| Ім'я для завдання 5 | new-wsrop.se-201-01.lab |

## Завдання 1. Складові URI

### 1.1. Розбір

<!-- Відсутню складову позначати «—». Порт за замовчуванням записувати з позначкою «(за замовч.)». -->

| № | scheme | userinfo | host | port | path | query | fragment |
|---|---|---|---|---|---|---|---|
| 1 | http | — | www.se-201-01.lab | 80 (за замовч.) | /reports/items | id=10&q=uk | section-2 |
| 2 | http | — | blog.se-201-01.lab | 80 | /items/ | id=%D0%BA%D0%B8%D1%97%D0%B2 | — |
| 3 | http | user:pass | edge.se-201-01.lab | 80 (за замовч.) | /reports | — | results |
| 4 | https | — | www.se-201-01.lab | 4443 | /reports/../items | — | — |
| 5 | mailto | — | — | — | admin@se-201-01.lab | subject=reports | — |

| № | Ім'я, що запитується в DNS | Заголовок `Host` | Ціль запиту (request target) |
|---|---|---|---|
| 1 | www.se-201-01.lab | www.se-201-01.lab | /reports/items?id=10&q=uk |
| 2 | blog.se-201-01.lab | blog.se-201-01.lab | /items/?id=%D0%BA%D0%B8%D1%97%D0%B2 |
| 3 | edge.se-201-01.lab | edge.se-201-01.lab | /reports   |
| 4 | www.se-201-01.lab | www.se-201-01.lab:4443 | /items |
| 5 | — | — | — |

Декодоване значення параметра URI 2:

Чому URI 5 не має складової host, хоча містить символ `@`:

### 1.2. Перевірка (після завдання 3)

<!-- Для URI 1–3: рядки запиту (починаються з ">") і тіло відповіді сервера. -->

```text
> GET /reports/items?id=10&q=uk HTTP/1.1
> Host: www.se-201-01.lab
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Sun, 15 Nov 2026 10:47:44 GMT
< Content-Type: text/plain
< Content-Length: 134
< Connection: keep-alive
< 
server: web-a
Host: www.se-201-01.lab
request URI: /reports/items?id=10&q=uk
path: /reports/items
query: id=10&q=uk
client: 10.53.1.1

> GET /items/?id=%D0%BA%D0%B8%D1%97%D0%B2 HTTP/1.1
> Host: blog.se-201-01.lab
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Sun, 15 Nov 2026 10:47:44 GMT
< Content-Type: text/plain
< Content-Length: 155
< Connection: keep-alive
< 
server: web-a
Host: blog.se-201-01.lab
request URI: /items/?id=%D0%BA%D0%B8%D1%97%D0%B2
path: /items/
query: id=%D0%BA%D0%B8%D1%97%D0%B2
client: 10.53.1.1

> GET /reports HTTP/1.1
> Host: edge.se-201-01.lab
> Authorization: Basic dXNlcjpwYXNz
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Sun, 15 Nov 2026 10:47:44 GMT
< Content-Type: text/plain
< Content-Length: 102
< Connection: keep-alive
< 
server: web-b
Host: edge.se-201-01.lab
request URI: /reports
path: /reports
query: 
client: 10.53.1.1
```

Значення заголовка `Authorization` і результат декодування:

```text
> Authorization: Basic dXNlcjpwYXNz
root@ns-se-201-01:/# echo 'dXNlcjpwYXNz' | base64 -d; echo
user:pass
```

URI 4: етап, на якому запит не вдається (пошук імені / TCP / TLS), і рядки виводу, що це підтверджують:

Host www.se-201-01.lab:4443 was resolved. - пошук імені в DNS був успішним.
connect to 10.53.0.80 port 4443 from 10.53.1.1 port 34648 failed: Connection refused - TCP-з'єднання не вийшло.

Порт 4443 на сервері закритий, тому сервер відхилив TCP-з'єднання і до TLS справа не дійшла.

```text
root@ns-se-201-01:/# curl -v 'https://www.se-201-01.lab:4443/reports/../items'
* Host www.se-201-01.lab:4443 was resolved.
* IPv6: (none)
* IPv4: 10.53.0.80
*   Trying 10.53.0.80:4443...
* connect to 10.53.0.80 port 4443 from 10.53.1.1 port 34648 failed: Connection refused
* Failed to connect to www.se-201-01.lab port 4443 after 41 ms: Couldn't connect to server
* Closing connection
curl: (7) Failed to connect to www.se-201-01.lab port 4443 after 41 ms: Couldn't connect to server
```

| Запит | `request URI`, отриманий сервером | `path`, отриманий сервером |
|---|---|---|
| звичайний | /items | /items |
| `--path-as-is` | /reports/../items | /items |

Розбіжності між URI та тим, що отримав сервер, і їх пояснення:
Пропали фрагменти #section-2 та #results, бо вони використовуються локально клієнтом і не передаються мережею. Зник порт у URI 2, бо порт 80 в HTTP використовується за замовчуванням, тому curl прибрав його з Host. Зникли дані користувача у URI 3, оскільки в Host має бути лише адреса сервера, тому curl переніс ці дані в заголовок Authorization. Шлях /reports/../items змінився на /items, оскільки curl самостійно скоротив сегменти з крапками. З прапорцем --path-as-is curl не чіпав шлях, але сервер скоротив його до /items.

## Завдання 2. Проходження ієрархії вручну

```text
root@ns-se-201-01:/# dig @10.53.0.2 ns.se-201-01.lab A +norec
dig @10.53.0.3 ns.se-201-01.lab A +norec
dig @10.53.1.1 ns.se-201-01.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.0.2 ns.se-201-01.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29321
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 2d5ff9c6a147f323010000006af8aee9da53061459aec36e (good)
;; QUESTION SECTION:
;ns.se-201-01.lab.              IN      A

;; AUTHORITY SECTION:
lab.                    172800  IN      NS      ns1.nic.lab.

;; ADDITIONAL SECTION:
ns1.nic.lab.            172800  IN      A       10.53.0.3

;; Query time: 80 msec
;; SERVER: 10.53.0.2#53(10.53.0.2) (UDP)
;; WHEN: Sat Nov 14 18:56:41 UTC 2026
;; MSG SIZE  rcvd: 117


; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.0.3 ns.se-201-01.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46173
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: dfc2d4b481d5cd26010000006af8aee9294e3678d2c5ce72 (good)
;; QUESTION SECTION:
;ns.se-201-01.lab.              IN      A

;; AUTHORITY SECTION:
se-201-01.lab.          3600    IN      NS      ns.se-201-01.lab.

;; ADDITIONAL SECTION:
ns.se-201-01.lab.       3600    IN      A       10.53.1.1

;; Query time: 80 msec
;; SERVER: 10.53.0.3#53(10.53.0.3) (UDP)
;; WHEN: Sat Nov 14 18:56:41 UTC 2026
;; MSG SIZE  rcvd: 103


; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 ns.se-201-01.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50330
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: a997468115571043010000006af8aee93996e1b0478f4be7 (good)
;; QUESTION SECTION:
;ns.se-201-01.lab.              IN      A

;; ANSWER SECTION:
ns.se-201-01.lab.       60      IN      A       10.53.1.1

;; AUTHORITY SECTION:
se-201-01.lab.          60      IN      NS      ns.se-201-01.lab.

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Sat Nov 14 18:56:41 UTC 2026
;; MSG SIZE  rcvd: 103
```

| Сервер | status | flags | ANSWER | AUTHORITY | ADDITIONAL |
|---|---|---|---|---|---|
| 10.53.0.2 | NOERROR | qr | — | lab. 172800 IN NS ns1.nic.lab. | ns1.nic.lab. 172800 IN A 10.53.0.3 |
| 10.53.0.3 | NOERROR | qr | — | se-201-01.lab. 3600 IN NS ns.se-201-01.lab. | ns.se-201-01.lab. 3600 IN A 10.53.1.1 |
| 10.53.1.1 | NOERROR | qr aa | ns.se-201-01.lab. 60 IN A 10.53.1.1 | se-201-01.lab. 60 IN NS ns.se-201-01.lab. | — |

Вивід `dig ns.sNN.lab A +trace`:

```text
root@ns-se-201-01:/# dig ns.se-201-01.lab A +trace

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> ns.se-201-01.lab A +trace
;; global options: +cmd
.                       71048   IN      NS      a.root-servers.net.
;; Received 59 bytes from 10.53.0.53#53(10.53.0.53) in 11 ms

lab.                    172800  IN      NS      ns1.nic.lab.
;; Received 117 bytes from 10.53.0.2#53(a.root-servers.net) in 83 ms

se-201-01.lab.          3600    IN      NS      ns.se-201-01.lab.
;; Received 103 bytes from 10.53.0.3#53(ns1.nic.lab) in 39 ms

ns.se-201-01.lab.       60      IN      A       10.53.1.1
se-201-01.lab.          60      IN      NS      ns.se-201-01.lab.
;; Received 103 bytes from 10.53.1.1#53(ns.se-201-01.lab) in 0 ms
```

| Запит | TTL запису NS | Прапорець `aa` | Секція, у якій міститься NS |
|---|---|---|---|
| `dig @10.53.0.3 sNN.lab NS +norec` | 3600 | відсутній | AUTHORITY SECTION |
| `dig @10.53.1.1 sNN.lab NS +norec` | 60 | присутній | ANSWER SECTION |

1. Які відповіді є відсиланнями і за якими ознаками це визначено?

2. Для чого у відповіді TLD присутній запис у секції ADDITIONAL і що сталося б, якби його не було?

3. Яка з двох копій запису NS є авторитативною і чому?

## Завдання 3. Заповнення власної зони

Вивід `zone-reload`:

```text
root@ns-se-201-01:/# zone-reload
zone se-201-01.lab/IN: loaded serial 2026110902
OK
zone reload queued
```

<!-- Шість запитів до 10.53.1.1 з +norec, по одному на кожен рядок таблиці картки. -->

| Запис | status | Прапорець `aa` | Значення в ANSWER | TTL | Відповідає картці (так/ні) |
|---|---|---|---|---|---|
| `www` A | NOERROR | присутній | www.se-201-01.lab. 60 IN A 10.53.0.80 | 60 | так |
| псевдонім CNAME | NOERROR | присутній | blog.se-201-01.lab. 60 IN CNAME www.se-201-01.lab. | 60 | так |
| `mail` A | NOERROR | присутній | mail.se-201-01.lab. 60 IN A 10.53.0.80 | 60 | так |
| `@` MX | NOERROR | присутній | se-201-01.lab. 60 IN MX 25 mail.se-201-01.lab. | 60 | так |
| `@` TXT | NOERROR | присутній | se-201-01.lab. 60 IN TXT "lab-token=3bc4ea6d" | 60 | так |
| другий хост A | NOERROR | присутній | edge.se-201-01.lab. 50 IN A 10.53.0.81  | 50 | так |

Порівняння відповідей резолвера та власного сервера:

```text
root@ns-se-201-01:/# dig blog.se-201-01.lab
dig @10.53.1.1 blog.se-201-01.lab +norec
dig se-201-01.lab MX
dig @10.53.1.1 se-201-01.lab MX +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> blog.se-201-01.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47377
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;blog.se-201-01.lab.            IN      A

;; ANSWER SECTION:
blog.se-201-01.lab.     60      IN      CNAME   www.se-201-01.lab.
www.se-201-01.lab.      60      IN      A       10.53.0.80

;; Query time: 39 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Sun Nov 15 09:49:42 UTC 2026
;; MSG SIZE  rcvd: 81


; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 blog.se-201-01.lab +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38954
;; flags: qr aa; QUERY: 1, ANSWER: 2, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 5ced50136f46e6cf010000006af98036675d5e0c04bae739 (good)
;; QUESTION SECTION:
;blog.se-201-01.lab.            IN      A

;; ANSWER SECTION:
blog.se-201-01.lab.     60      IN      CNAME   www.se-201-01.lab.
www.se-201-01.lab.      60      IN      A       10.53.0.80

;; AUTHORITY SECTION:
se-201-01.lab.          60      IN      NS      ns.se-201-01.lab.

;; ADDITIONAL SECTION:
ns.se-201-01.lab.       60      IN      A       10.53.1.1

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Sun Nov 15 09:49:42 UTC 2026
;; MSG SIZE  rcvd: 142


; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> se-201-01.lab MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63793
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;se-201-01.lab.                 IN      MX

;; ANSWER SECTION:
se-201-01.lab.          60      IN      MX      25 mail.se-201-01.lab.

;; Query time: 0 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Sun Nov 15 09:49:42 UTC 2026
;; MSG SIZE  rcvd: 63


; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 se-201-01.lab MX +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61359
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 7bc6d1438e1c7ac8010000006af980365724aba2454dd3ed (good)
;; QUESTION SECTION:
;se-201-01.lab.                 IN      MX

;; ANSWER SECTION:
se-201-01.lab.          60      IN      MX      25 mail.se-201-01.lab.

;; AUTHORITY SECTION:
se-201-01.lab.          60      IN      NS      ns.se-201-01.lab.

;; ADDITIONAL SECTION:
mail.se-201-01.lab.     60      IN      A       10.53.0.80
ns.se-201-01.lab.       60      IN      A       10.53.1.1

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Sun Nov 15 09:49:42 UTC 2026
;; MSG SIZE  rcvd: 140
```

| Запит | flags | ANSWER (кількість, типи) | AUTHORITY (кількість, типи) | ADDITIONAL (кількість, типи) | TTL |
|---|---|---|---|---|---|
| псевдонім через resolver-a | qr rd ra | 2 CNAME, A | 0 | 0 | 60 |
| псевдонім через 10.53.1.1 | qr aa | 2 CNAME, A | 1 NS | 1 A | 60 |
| MX через resolver-a | qr rd ra | 1 MX | 0 | 0 | 60 |
| MX через 10.53.1.1 | qr aa | 1 MX | 1 NS | 2 A, A | 60 |

Пояснення відмінностей:

Повідомлення `zone-check` після тимчасового додавання `@ IN CNAME www`:

```text
root@ns-se-201-01:/# zone-check
dns_master_load: /lab/zone/se-201-01.lab.zone:22: se-201-01.lab: CNAME and other data
zone se-201-01.lab/IN: loading from master file /lab/zone/se-201-01.lab.zone failed: CNAME and other data
zone se-201-01.lab/IN: not loaded due to errors.
```

1. Який TTL має кожен запис зони і звідки це значення взято?

2. Скільки записів містить секція ANSWER у відповіді на запит псевдоніма і чому?

3. Чому запис CNAME неможливо розмістити для імені `@`?

## Завдання 4. TTL і видимість змін

Два запити з інтервалом 5 с:

```text
root@ns-se-201-01:/# dig www.se-201-01.lab +noall +answer; sleep 5; dig www.se-201-01.lab +noall +answer
www.se-201-01.lab.      60      IN      A       10.53.0.80
www.se-201-01.lab.      55      IN      A       10.53.0.80
```

**Прогноз** (записати до вимірювання). Формула часу появи нової адреси через залишок R, найбільша можлива затримка: час зміни + R = очікуванний час. 60 С

Перевірка, що сервер до `zone-reload` повертає стару адресу:

```text
root@ns-se-201-01:/# dig @10.53.1.1 www.se-201-01.lab +norec +short
10.53.0.80
```

Залишок R і час (крок 4), час `zone-reload` (крок 5), перевірка на сервері (крок 6):

```text
root@ns-se-201-01:/# date +%T; dig www.se-201-01.lab +noall +answer
15:19:58
www.se-201-01.lab.      37      IN      A       10.53.0.80
root@ns-se-201-01:/# zone-reload; date +%T; dig @10.53.1.1 www.se-201-01.lab +norec +short
zone se-201-01.lab/IN: loaded serial 2026110906
OK
zone reload queued
15:20:02
10.53.0.81
```

Очікуваний час появи нової адреси (підставити R у формулу): 
15:19:58 + 37 с = 15:20:35

Вивід циклу спостереження (перші два рядки, два останні рядки зі старою адресою, рядок з новою адресою):

```text
root@ns-se-201-01:/# while true; do echo "$(date +%T) $(dig +short www.se-201-01.lab)"; sleep 2; done
15:20:02 10.53.0.80
15:20:04 10.53.0.80
...
15:20:33 10.53.0.80
15:20:35 10.53.0.80
15:20:37 10.53.0.81
^C
```

Рядок `server:` з відповіді `curl`:

| Час зміни | Очікуваний час | Спостережений час | Виміряна затримка, с | Прогноз, с |
|---|---|---|---|---|
| 15:20:02 | 15:20:35 | 15:20:37 | 35 | 37 |

1. Чим пояснюється розбіжність між прогнозом і виміряним значенням?

2. Якою була б найбільша можлива затримка для цього запису і за яких умов?

3. Як слід діяти, щоб перенести сервіс на нову адресу, якщо TTL запису становить 86 400 с?

## Завдання 5. Негативне кешування

**Прогноз** (записати до вимірювання). Формула за RFC 2308 і розраховане значення: TTL негативного кешування = min(TTL SOA, minimum SOA) = min(20, 90) = 20

Перевірка, що сервер до `zone-reload` повертає `NXDOMAIN`:

```text
root@ns-se-201-01:/# dig @10.53.1.1 new-wsrop.se-201-01.lab +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.9-Ubuntu <<>> @10.53.1.1 new-wsrop.se-201-01.lab +norec
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 5830
;; flags: qr aa; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
;; AUTHORITY SECTION:
se-201-01.lab.          20      IN      SOA     ns.se-201-01.lab. admin.se-201-01.lab. 2026110906 3600 600 86400 90
```

Запит через resolver-a (крок 3), час `zone-reload` (крок 4), перевірка на сервері (крок 5):

```text
root@ns-se-201-01:/# date +%T; dig new-wsrop.se-201-01.lab
zone-reload; date +%T
dig @10.53.1.1 new-wsrop.se-201-01.lab +norec +short
16:33:59
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 49103
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
;; AUTHORITY SECTION:
se-201-01.lab.          20      IN      SOA     ns.se-201-01.lab. admin.se-201-01.lab. 2026110906 3600 600 86400 90
zone se-201-01.lab/IN: loaded serial 2026110907
OK
zone reload queued
16:33:59
10.53.0.80
```

Вивід циклу спостереження (перші два рядки, останній рядок з `NXDOMAIN`, перший рядок з `NOERROR`):

```text
root@ns-se-201-01:/# while true; do echo "$(date +%T) $(dig new-wsrop.se-201-01.lab +noall +comments | grep -o 'status: [A-Z]*')"; sleep 2; done
16:34:04 status: NXDOMAIN
16:34:06 status: NXDOMAIN
...
16:34:18 status: NXDOMAIN
16:34:20 status: NOERROR
^C
```

| Негативну відповідь закешовано | Запис створено | Ім'я резолвиться через resolver-a | Виміряно, с | Прогноз, с |
|---|---|---|---|---|
| 16:33:59 | 16:33:59 | 16:34:20 | 21 | 20 |

<!-- Якщо завдання повторювалося з іншим ім'ям (new2-…), зазначити це й причину. -->

1. Чому тривалість негативного кешування визначається записом SOA, а не записом, який запитували?

2. Чи збігається TTL запису SOA в секції AUTHORITY з TTL запису SOA у файлі зони? Поясніть.

3. Скрипт розгортання перевіряє ім'я сервісу до створення запису, а потім одразу після. Що він побачить і як це виправити?

## Завдання 6. Затримка пошуку імені

**Прогноз** (записати до вимірювання). До яких серверів звернеться resolver-a:

| Ім'я | Перший запит | Повторний запит |
|---|---|---|
| A: нове ім'я у власній зоні | | |
| B: нове ім'я в зоні `lab.` | | |
| C: ім'я в неіснуючому TLD | | |
| D: `www.sNN.lab` після перезапуску резолвера | | |

Вивід вимірювань:

```text

```

| Ім'я | Перший запит, мс | Повторний, мс | Сервери, до яких звернувся резолвер (за журналами) |
|---|---|---|---|
| A | | | |
| B | | | |
| C | | | |
| D | | | |

Значення `time_namelookup` з `curl` після перезапуску резолвера:

Фрагменти журналів запитів root, TLD і власного сервера для рядка D:

```text

```

1. Яку затримку вносить кожен із серверів root, TLD і власний сервер? Покажіть, з яких рядків таблиці це випливає.

2. Чому в рядку A перший запит не проходить через root і TLD, хоча ім'я нове?

3. Вебсторінка завантажує ресурси з п'яти різних доменів. Як це впливає на час першого відкриття сторінки?

## Завдання 7. Інцидент: зупинка власного сервера

**Прогноз** (записати до вимірювання). Що поверне кожен резолвер після зупинки сервера та спливання TTL і скільки часу займе відповідь:

Кешування запису в обох резолверах (крок 2) і час зупинки сервера (крок 3):

```text

```

Запити із заміром часу (крок 5):

```text

```

| Запит | status або тайм-аут | ANSWER, TTL | `real` |
|---|---|---|---|
| resolver-a, перший | | | |
| resolver-a, другий | | | |
| resolver-b | | | |

Час `named-start` і вивід циклу спостереження до першої успішної відповіді (крок 6):

```text

```

Перевірка після завдання (`dig @10.53.1.1 sNN.lab SOA +norec +short`):

```text

```

### Постмортем

1. Що сталося:
2. Вплив на клієнтів resolver-a і resolver-b:
3. Хронологія з виміряним часом:
4. Чому кеш допоміг або не допоміг:
5. Що слід змінити:

1. Який TTL мав запис у відповіді resolver-b і навіщо такий TTL?

2. Порівняйте результат і тривалість двох запитів до resolver-a. Якщо вони різняться, сформулюйте гіпотезу щодо причини; якщо ні, поясніть, що робить резолвер протягом цього часу.

3. Чому після `named-start` ім'я через resolver-a зарезолвилося не одразу?

4. У якій ситуації serve-stale шкодить, а не допомагає? Пов'яжіть відповідь із завданням 4.

## Фінальний файл зони

<!-- Повний вміст stand/students/sNN/zone/sNN.lab.zone після завершення роботи. -->

```text

```

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено (RFC, документація, додатковий експеримент на стенді) і що з'ясувати не вдалося:

## Декларація використання ШІ

Використовувалися інструменти ШІ: <!-- так / ні -->

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | | | |
