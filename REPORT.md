# Отчёт по домашней работе

Делал на WSL2 (Ubuntu 26.04), systemd запущен как PID 1. Стенд поднял через `sudo ./scripts/setup.sh`.

## Перед началом: порт 8080 был занят

Перед запуском решил проверить что порт свободн:

```bash
sudo ss -lntp | grep 8080
# LISTEN 0  4096  *:8080  *:*
```

Процесс не показывался даже под sudo, значит сокет держит не процесс внутри WSL. Через `docker ps` нашёл что 8080 пробрасывает контейнер nginx из другого моего проекта (`wa2-23pavel23-nginx-1`, `0.0.0.0:8080->80/tcp`). Остановил его в Docker Desktop, после этого `ss` ничего на 8080 не показывал. Иначе приложение просто не смогло бы занять порт.

---

## Проблема 1: неправильный ExecStart

### Что было обнаружено

```text
Active: activating (auto-restart) (Result: exit-code)
Process: 737 ExecStart=/opt/linux-devops-homework/app.py (code=exited, status=203/EXEC)
```

Сервис постояно перезапускается с кодом 203/EXEC. Это ошибка systemd, а не приложения: он не смог запустить файл из ExecStart.

### Как проводилась диагностика

```bash
systemctl status homework-app.service --no-pager
systemctl cat homework-app
systemctl show homework-app -p ExecStart --value
journalctl -u homework-app -n 10 --no-pager
ls -l /opt/linux-devops-homework
```

В журнале было `Unable to locate executable '/opt/linux-devops-homework/app.py': No such file or directory`. Через `systemctl show` увидел точный путь который использует systemd, а `ls` показал что в каталоге лежит только `server.py`.

### В чём была причина

ExecStart указывает на файл `app.py` которого не существует.

### Что было изменено

В `systemd/homework-app.service`:

```diff
-ExecStart=/opt/linux-devops-homework/app.py
+ExecStart=/opt/linux-devops-homework/app/server.py
```

```bash
sudo install -d -m 0755 /opt/linux-devops-homework/app
sudo mv /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py
sudo install -m 0644 systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
sudo systemctl restart homework-app
```

### Почему было выбрано это решение

Путь `app/server.py` такой же как в репозитории, и именно его ожидает проверка, поэтому перенёс файл в подкаталог `app/` и поправил unit. Unit правил в репозитории и копировал в /etc, чтобы в гите и в системе было одно и тоже. `daemon-reload` нужен потому что иначе systemd продолжит использовать старую верию unit-а.

### Как проверялся результат

```bash
systemctl show homework-app -p ExecStart --value
journalctl -u homework-app -n 15 --no-pager
```

ExecStart стал правильный, 203/EXEC пропал. Теперь в журнале traceback питона, то есть приложение запускается, но падает уже само.

---

## Проблема 2: нет прав на каталог данных

### Что было обнаружено

```text
PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'
homework-app.service: Main process exited, code=exited, status=1/FAILURE
```

Кстати `systemctl status` сразу после рестарта показвал `active (running)`, но это было через 100мс после старта, через секунду он снова падал. Поэтому смотрел журнал.

### Как проводилась диагностика

```bash
id homework
namei -l /var/lib/linux-devops-homework
stat /var/lib/linux-devops-homework
```

```text
uid=999(homework) gid=986(homework) groups=986(homework)
drwx------ root root linux-devops-homework
Access: (0700/drwx------)  Uid: (0/root)  Gid: (0/root)
```

### В чём была причина

Сервис работает от `homework`, а каталог принадлежит `root:root` с правами 0700. У homework нет прав ни зайти в катлог, ни создать там `startup.log`.

### Что было изменено

```bash
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 0750 /var/lib/linux-devops-homework
sudo systemctl restart homework-app
```

### Почему было выбрано это решение

Владельцу (homework) нужны rwx чтобы писать лог, группе хватит чтения, остальным достп не нужен. chmod 777 и запуск от root запрещены и это небезопасно. Менять права только на сам файл не вариант, без прав на каталог к нему не добраться.

### Как проверялся результат

```bash
stat -c '%U:%G %a %n' /var/lib/linux-devops-homework
# homework:homework 750 /var/lib/linux-devops-homework
systemctl status homework-app --no-pager
# Active: active (running) since ... 7s ago
sudo cat /var/lib/linux-devops-homework/startup.log
# starting on 127.0.0.1:8080
```

Сервис стабильно работает, лог записалься. Но в логе видно что он слушает на 127.0.0.1.

---

## Проблема 3: слушает только на 127.0.0.1

### Что было обнаружено

```text
LISTEN 0  5  127.0.0.1:8080  0.0.0.0:*  users:(("python3",pid=1622,fd=3))
```

### Как проводилась диагностика

```bash
sudo ss -lntp | grep 8080
cat /etc/linux-devops-homework/app.conf
cat config/app.conf
```

В обоих конфигах `host = 127.0.0.1` стояло.

### В чём была причина

127.0.0.1 это loopback, на такой адрес можно подключиться только с этой же машины. 0.0.0.0 значит слушать на всех IPv4 интерфейсах, тогда сервис доступен и снаружи (через eth0) и локально через 127.0.0.1.

### Что было изменено

В `config/app.conf`:

```diff
-host = 127.0.0.1
+host = 0.0.0.0
```

```bash
sudo install -m 0644 config/app.conf /etc/linux-devops-homework/app.conf
sudo systemctl restart homework-app
```

### Почему было выбрано это решение

server.py менять нельзя, адрес берётся из конфига. Конфиг читаеться только при старте поэтому нужен рестарт.

### Как проверялся результат

```bash
sudo ss -lntp | grep 8080
# LISTEN 0  5  0.0.0.0:8080  0.0.0.0:*  users:(("python3",pid=1864,fd=3))
curl -v http://127.0.0.1:8080/health
# < HTTP/1.0 200 OK
# {"status": "ok"}
```

---

## Дополнительные наблюдения

```bash
systemctl is-enabled homework-app   # enabled (сделал setup.sh)
PID=$(systemctl show homework-app -p MainPID --value)   # 1864
ps -o pid,ppid,user,group,args -p $PID
#   1864   1 homework homework python3 /opt/linux-devops-homework/app/server.py
grep -E '^(Name|State|Pid|PPid|Uid|Gid)' /proc/$PID/status
# PPid: 1
# Uid:  999 999 999 999
# Gid:  986 986 986 986
tr '\0' ' ' < /proc/$PID/cmdline
# python3 /opt/linux-devops-homework/app/server.py
sudo ls -l /proc/$PID/fd/
# 0 -> /dev/null
# 1 -> socket:[98464]
# 2 -> socket:[98464]
# 3 -> socket:[101927]
ip route get 1.1.1.1
# 1.1.1.1 via 172.17.80.1 dev eth0 src 172.17.85.154
```

- PID 1864, PPID 1, т.е. процесс запустил systemd.
- В `/proc/PID/status` все UID = 999 (homework), значит процесс работает не от root.
- В cmdline видно что запущен python3 с нашим server.py (через shebang).
- В fd: 0 это /dev/null, 1 и 2 это сокет в journald (поэтому вывод видно в journalctl), 3 это слушающий сокет, тот же fd=3 что показывает `ss`.
- До 1.1.1.1 трафик пойдёт через дефолтный шлюз 172.17.80.1 через eth0 с адресом 172.17.85.154.

## Итоговый ход диагностики

Сначала освбодил порт 8080 от докер контейнера. Потом `systemctl status` показал 203/EXEC, исправил ExecStart и перенёс server.py в app/. После этого приложение стало запускатся но падало с PermissionError, поправил владельца и права на /var/lib/linux-devops-homework. Потом сервис заработал, но слушал на 127.0.0.1, поменял host на 0.0.0.0 в конфиге. Каждая следующая ошибка была видна только после исправления предыдущей.

Системные команды которых нет в гите: `mkdir`/`mv` для server.py, `install` unit-а и конфига в /etc, `daemon-reload`, `chown`/`chmod` каталога данных, `restart`. `enable` сделал setup.sh.

## Итоговая проверка

```bash
sudo ./scripts/check.sh
```

```text
Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed
```
