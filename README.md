# Домашнее задание к занятию "`2 «Кластеризация и балансировка нагрузки»`" - `Юрочкин Виктор`

### Задание 1
Запустите два simple python сервера на своей виртуальной машине на разных портах
Установите и настройте HAProxy, воспользуйтесь материалами к лекции по ссылке
Настройте балансировку Round-robin на 4 уровне.
На проверку направьте конфигурационный файл haproxy, скриншоты, где видно перенаправление запросов на разные серверы при обращении к HAProxy.

`**Решение**`

1. `На своём домашнем (сервере) гипервизове proxmox, "поднял" виртуалку с ubuntu server 22.04`
<img width="484" height="771" alt="1" src="https://github.com/user-attachments/assets/621b4f91-2c6d-4170-a7b4-1b4d7de46404" />

<img width="1215" height="954" alt="2" src="https://github.com/user-attachments/assets/44208b07-2591-4804-bcaf-7a1e6a64a090" />

2. `Установил все необходимые обновления и пакеты:`
```
sudo apt update
sudo apt install -y nano python3 haproxy
```

3. `Создаю две директории и разные страницы`
```
sudo mkdir -p /var/www/py1 /var/www/py2
echo "Hello from PY1" | sudo tee /var/www/py1/index.html
echo "Hello from PY2" | sudo tee /var/www/py2/index.html
```

4. `Запускаем два простых HTTP-сервера
В первой вкладке/сессии:`

```
cd /var/www/py1
python3 -m http.server 8001
```
<img width="746" height="307" alt="3" src="https://github.com/user-attachments/assets/6a03728e-8bd1-4089-983e-0fa625efcc40" />

`Во второй вкладке/сессии:`
```
cd /var/www/py2
python3 -m http.server 8002
```
<img width="746" height="144" alt="4" src="https://github.com/user-attachments/assets/1eab4335-1155-45db-83be-f76bd5c91312" />

5. `Проверяю, что "поднялось:`
```
curl http://127.0.0.1:8001
curl http://127.0.0.1:8002

```
<img width="468" height="190" alt="5" src="https://github.com/user-attachments/assets/2810f272-b52a-4eb1-a24b-365128299d19" />

6. `Конфигурация HAProxy (L4, round-robin)`

```
nano /etc/haproxy/haproxy.cfg
```
```
global
    log /dev/log local0
    log /dev/log local1 notice
    user haproxy
    group haproxy
    daemon
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s

defaults
    log     global
    mode    tcp               # L4 (TCP)
    option  dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s

# Веб-страница (статистика)
listen stats
    mode http
    bind :888
    stats enable
    stats uri /stats
    stats refresh 5s
    stats realm Haproxy\ Statistics

# TCP-балансировка round-robin между двумя Python-серверами
listen web_tcp
    mode tcp
    bind :1325
    balance roundrobin
    option tcp-check

    server s1 127.0.0.1:8001 check inter 3s
    server s2 127.0.0.1:8002 check inter 3s
```
`Проверка синтаксиса`
```
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```
`Если всё ок`
`Перезапустим службу`
```
sudo systemctl restart haproxy
sudo systemctl status haproxy
```
7. `Проверка балансировки`

`Для наглядности выполню команду несколько раз:`
```
curl http://127.0.0.1:8080
curl http://127.0.0.1:8080
curl http://127.0.0.1:8080
curl http://127.0.0.1:8080
```
`Так же в конфиге я учел статистику, что бы посмотреть, что это действительно работает и вызовы чередуются`
```
curl http://127.0.0.1:1325
```
<img width="483" height="604" alt="6" src="https://github.com/user-attachments/assets/71451383-37aa-4369-a493-4347c05019af" />

---

### Задание 2 Запустите три simple python сервера на своей виртуальной машине на разных портах
Настройте балансировку Weighted Round Robin на 7 уровне, чтобы первый сервер имел вес 2, второй - 3, а третий - 4 HAproxy должен балансировать только тот http-трафик, который адресован домену example.local На проверку направьте конфигурационный файл haproxy, скриншоты, где видно перенаправление запросов на разные серверы при обращении к HAProxy c использованием домена example.local и без него.

`**Решение**`
`делаем на той же VM`

1. `Три simple Python-сервера
Сделаем порты 8101, 8102, 8103`
```
mkdir -p /var/www/py8101 /var/www/py8102 /var/www/py8103
echo "Hello from PY8101" | sudo tee /var/www/py8101/index.html
echo "Hello from PY8102" | sudo tee /var/www/py8102/index.html
echo "Hello from PY8103" | sudo tee /var/www/py8103/index.html
```
<img width="889" height="221" alt="7" src="https://github.com/user-attachments/assets/c161472c-c86c-452a-802b-8fde7d88d7f0" />

2. `Запускаем 3 сервера
Открой 3 терминала/сессии (или tmux/screen).
Терминал 1:`
```
cd /var/www/py8101
python3 -m http.server 8101
```
<img width="739" height="110" alt="8" src="https://github.com/user-attachments/assets/43587ed5-306e-48ea-ab18-53023b196c27" />
`Терминал 2:`
```
cd /var/www/py8102
python3 -m http.server 8102
```
<img width="739" height="110" alt="9" src="https://github.com/user-attachments/assets/f4e02472-8359-4243-a83a-64b9388135b5" />
`Терминал 3:`
```
cd /var/www/py8103
python3 -m http.server 8103
```
<img width="739" height="110" alt="10" src="https://github.com/user-attachments/assets/9235358d-2c4a-44ea-8e5f-f642d079432d" />
`Проверим, что поднялись`
```
curl http://127.0.0.1:8101
curl http://127.0.0.1:8102
curl http://127.0.0.1:8103
```
<img width="457" height="170" alt="11" src="https://github.com/user-attachments/assets/befc4654-5ea2-4ac6-a3fd-929d753105df" />

3. `Доменное имя example.local
Чтобы HAProxy мог отфильтровать трафик по домену, добавим его в /etc/hosts`
```
echo "127.0.0.1 example.local" | sudo tee -a /etc/hosts
```
`и сразу проверим, что он отвечает`
```
ping -c 1 example.local
```
<img width="848" height="234" alt="12" src="https://github.com/user-attachments/assets/4a867cdb-de37-443a-9ac3-9c29e8de9c74" />


5. `Настройка HAProxy — Weighted Round Robin на L7
Я дополню свой текущий /etc/haproxy/haproxy.cfg, не ломая задание 1.
Мы уже знаем, что там есть global, defaults, listen stats, listen web_tcp. Оставляем всё как есть и ниже добавляем новые секции frontend и backend.`
```
nano /etc/haproxy/haproxy.cfg
```
`добавляем в самый низ файла`
```
frontend fe_example_local
    mode http
    bind *:8080

    acl host_example_local hdr_beg(host) -i example.local
    use_backend be_example_local if host_example_local

backend be_example_local
    mode http
    balance roundrobin

    # Weighted Round Robin: веса 2, 3, 4
    server py1 127.0.0.1:8101 weight 2 check
    server py2 127.0.0.1:8102 weight 3 check
    server py3 127.0.0.1:8103 weight 4 check
```
`проверим конфиг и перезапустим службу`
```
haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl status haproxy
```
<img width="1439" height="510" alt="13" src="https://github.com/user-attachments/assets/9b6abec5-fbb5-4262-a5c6-af32bfd2b3b1" />

6. `Проверка балансировки:`
```
curl http://example.local:8080/
curl http://example.local:8080/
curl http://example.local:8080/
curl http://example.local:8080/
curl http://example.local:8080/
```
`без домена`
```
curl http://127.0.0.1:8080/
```
<img width="601" height="349" alt="14" src="https://github.com/user-attachments/assets/74c6a1e9-8c30-4de0-b80f-f3da86a9e0e7" />
