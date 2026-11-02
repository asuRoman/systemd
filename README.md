Задание 1: Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова

создать файл в /etc/default с конфигурацией сервиса

touch /etc/default/testlog

добавить в файл строки:
WORD="ALERT"
LOG=/var/log/testlog.log

создаем файл для лога и записываем туда строки и ключевое словао 'ALERT'

touch /var/log/testlog.log

создадим скрипт выполнения в файле /opt/testlog.sh

touch /opt/testlog.sh

и закинем сам скрипт в этот файл:

#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: Test Asur!"
else
exit 0
fi

добавим права на запуск нашего скрипта

chmod +x /opt/testlog.sh

создадим юнит для сервиса

cat > /etc/systemd/system/testlog.service
[Unit]
Description=testlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/testlog
ExecStart=/opt/testlog.sh $WORD $LOG

создадим юнит для таймера

cat > /etc/systemd/system/testlog.timer
[Unit]
Description=Run watchlog script every 30 second

[Timer]
Run every 30 second
OnUnitActiveSec=30
Unit=testlog.service

[Install]
WantedBy=multi-user.target

Запускаем таймер

systemctl start watchlog.timer

Задание 2: Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта

для начала устанавливаем spawn-fcgi и все необходимые пакеты

apt install spawn-fcgi php php-cgi php-cli apache2 libapache2-mod-fcgid -y
 
 создаем файл с настройками для будущего сервиса и сам юнит-файл
 
 cat > /etc/spawn-fcgi/fcgi.conf
You must set some working options before the "spawn-fcgi" service will work.
If SOCKET points to a file, then this file is cleaned up by the init script.
See spawn-fcgi(1) for all possible options.
Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"


cat > /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target


запускаем и проверяем его работу

systemctl start spawn-fcgi

systemctl status spawn-fcgi

Задание 3: Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными 
конфигурационными файлами одновременно

для начала устанавливаем NGINX

apt install nginx -y

создаем юнит файл для работы с шаблонами (для запуска нескольких экземпляров сервиса)

cat > /etc/systemd/system/nginx@.service

[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target

создаем два файла конфигурации NGINX, я скопировал стандартный файл /etc/nginx/nginx.conf
и добавил в него, чтобы они слушали разные порты и разные pid были у них

далее запускаем сервисы и проверяем их статусы

systemctl start nginx@first
systemctl start nginx@second

чтобы проверить работоспособность сервисов я и спользовал команду

ss -tnulp | grep nginx

и получился такой вывод:

root@clean:/home/asur# ss -tnulp | grep nginx
tcp   LISTEN 0      511                 0.0.0.0:9002      0.0.0.0:*    users:(("nginx",pid=10916,fd=5),("nginx",pid=10915,fd=5),("nginx",pid=10914,fd=5))                                                   
tcp   LISTEN 0      511                 0.0.0.0:9001      0.0.0.0:*    users:(("nginx",pid=10907,fd=5),("nginx",pid=10906,fd=5),("nginx",pid=10905,fd=5))
