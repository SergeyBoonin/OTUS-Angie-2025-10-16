# Домашняя работа OTUS-Angie-2025-10-16 OTUS_Angie_16_Балансировка_нагрузки_(HTTP)

### 1. Создайте несколько копий бэкендов.

Один из четырёх бэкендов был настроен создавать задержку обработки в 2 секунды.

В Angie были созданы отдельные локации для каждого режима балансировки, использующие индивидуальные апстрим-группы, настроенные соответствующим образом:

| Локация            | Особенность                                                                            |
|--------------------|----------------------------------------------------------------------------------------|
| /                  | default round-robin with slow_start                                                    |
| /least_conn        | least_conn with slow_start                                                             |
| /random            | random                                                                                 |
| /hash              | hash $scheme$request_uri                                                               |
| /hash_keepalive    | то же плюс keepalive плюс http1.1 и удаление заголовка 'connection' в сторону апстрима |
| /rr_sticky_cookie  | default round-robin with slow_start плюс sticky_cookie                                 |


### 2. Настройте балансировку по следующим вариантам:

#### Равномерная балансировка (round robin).
<img width="1496" height="284" alt="rr" src="https://github.com/user-attachments/assets/6f110e48-7f35-42fb-8226-e86fc388ce1f" />

```
root@ubuntu01:/home/user/docker-ballance# wrk -t 10 -c 10 -d 10s http://192.168.2.115/rr
Running 10s test @ http://192.168.2.115/rr
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    18.67ms   13.58ms  60.27ms   79.49%
    Req/Sec    20.77     18.83    60.00     46.15%
  43 requests in 10.08s, 446.88KB read
  Socket errors: connect 0, read 0, write 0, timeout 4
Requests/sec:      4.26
Transfer/sec:     44.32KB
```

#### Балансировка по хэшу с использованием переменных (на выбор).

Не очень понимаю, как компактно показать результат тестирования. :)

Могу с уверенностью сказать, что режим **'hash $scheme$request_uri'** работал как ожидалось.

*В то же время заставить Angie использовать **keepalive** соединение в сторону апстрима не удалось: несмотря на настройку keepalive, использование HTTP1.1, и удаление заголовка **'connection'**, к тому же серверу каждый раз устатавливалось новое TCP соединение. Буду рад комментарию на этот счёт.*

#### Произвольная балансировка (random).
<img width="1508" height="286" alt="random" src="https://github.com/user-attachments/assets/9d438984-bdad-49e6-8a40-4eafafaee8fa" />

```
root@ubuntu01:/home/user/docker-ballance# wrk -t 10 -c 10 -d 10s http://192.168.2.115/random
Running 10s test @ http://192.168.2.115/random
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    22.06ms   16.58ms  68.00ms   74.07%
    Req/Sec    20.68     15.11    60.00     63.16%
  58 requests in 10.09s, 603.97KB read
  Socket errors: connect 0, read 0, write 0, timeout 4
Requests/sec:      5.75
Transfer/sec:     59.89KB
```

**Примечательно**, что в условиях существования одного сервера, испытывающего проблемы с задержкой обработки вызовов, результат режима **least_conn** оказался ощутимо лучше.
<img width="1512" height="285" alt="least_conn" src="https://github.com/user-attachments/assets/65181c25-a2d2-42f4-ba59-d03421ffd111" />

```
root@ubuntu01:/home/user/docker-ballance# wrk -t 10 -c 10 -d 10s http://192.168.2.115/least_conn
Running 10s test @ http://192.168.2.115/least_conn
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    51.40ms   60.12ms 514.94ms   89.64%
    Req/Sec    26.71     13.27    60.00     49.09%
  1706 requests in 10.08s, 17.38MB read
  Socket errors: connect 0, read 0, write 0, timeout 1
Requests/sec:    169.29
Transfer/sec:      1.72MB
```

### 3. Покажите варианты конфигурации с резервным бэкендом и с отключением одного из бэкендов. Протестируйте корректность настроенных схем.

С помощью команды **docker stop debug-blue** в процессе тестирования выключался один из серверов.

Во всех случаях количество потеряных вызовов, вызванных активной недоступностью одного сервера, было адекватным (~5 запросов)

<img width="1497" height="283" alt="rr" src="https://github.com/user-attachments/assets/afec3c58-2152-4bde-a045-e95ce3d54c79" />

```
root@ubuntu01:/home/user# wrk -t 10 -c 10 -d 10s http://192.168.2.115/rr
Running 10s test @ http://192.168.2.115/rr
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    21.88ms   22.38ms  89.30ms   87.50%
    Req/Sec    18.29     19.70    70.00     82.35%
  45 requests in 10.07s, 467.66KB read
  Socket errors: connect 0, read 0, write 0, timeout 5
Requests/sec:      4.47
Transfer/sec:     46.44KB
```

<img width="1506" height="290" alt="random" src="https://github.com/user-attachments/assets/dfea796d-e707-4b1a-bb14-29f1465291c9" />

```
root@ubuntu01:/home/user# wrk -t 10 -c 10 -d 10s http://192.168.2.115/random
Running 10s test @ http://192.168.2.115/random
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    30.17ms   33.72ms 114.65ms   86.84%
    Req/Sec    16.47     19.64    80.00     84.21%
  43 requests in 10.08s, 447.72KB read
  Socket errors: connect 0, read 0, write 0, timeout 5
Requests/sec:      4.27
Transfer/sec:     44.43KB
```

<img width="1510" height="282" alt="least_conn" src="https://github.com/user-attachments/assets/c328ca77-ebcd-45c4-8ea2-1b0c7af5186b" />

```
root@ubuntu01:/home/user# wrk -t 10 -c 10 -d 10s http://192.168.2.115/least_conn
Running 10s test @ http://192.168.2.115/least_conn
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    43.21ms   65.65ms 828.47ms   93.19%
    Req/Sec    31.77     13.46    70.00     72.16%
  1851 requests in 10.06s, 18.86MB read
  Socket errors: connect 0, read 0, write 0, timeout 4
Requests/sec:    184.03
Transfer/sec:      1.87MB
```

Так же переключение успешно состоялось и при использовании режима hash. Выключалась нода, выбранная хэшем.

<img width="1517" height="279" alt="image" src="https://github.com/user-attachments/assets/492c3720-3457-4da4-9c7d-f02095288d96" />

```
root@ubuntu01:/home/user# wrk -t 10 -c 10 -d 10s http://192.168.2.115/hash/777
Running 10s test @ http://192.168.2.115/hash/777
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    84.22ms   76.51ms 466.63ms   86.68%
    Req/Sec    16.95      7.62    30.00     71.09%
  1478 requests in 10.05s, 15.03MB read
Requests/sec:    147.06
Transfer/sec:      1.50MB
```
### 4. Настройте балансировку с использованием методов sticky route или stricky cookie, покажите работоспособность конфигурации на стенде.

Настроено. Работает как ожидается во всех сценариях.

Поскольку wrk не поддерживает cookies (или я не знаю, как его заставить), иллюстраций не прикладываю.
