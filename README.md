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

В то же время заставить Angie использовать **keepalive** соединение в сторону апстрима не удалось: несмотря на настройку keepalive, использование HTTP1.1, и удаление заголовка **'connection'**, к тому же серверу каждый раз устатавливалось новое TCP соединение.

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
