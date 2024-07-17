# **Проект**

Итоговый проект курса [Administrator Linux.Professional](https://otus.ru/lessons/linux-professional/)

Создание рабочего проекта
веб проект с развертыванием нескольких виртуальных машин должен отвечать следующим требованиям:

- включен https;
- основная инфраструктура в DMZ-зоне;
- firewall на входе;
- сбор метрик и настроенный alerting;
- организован централизованный сбор логов.

# **Системные требования**

Для развёртывания 7 виртуальных машин (ВМ) требуется 14 ядер процессора и 10 ГБ оперативной памяти. На HDD ~весь этот колхоз~ этот проект разворачивается 80-90 минут. На SSD должно быть быстрее. Самое долгое - около 10 минут на каждую ВМ - обновление пакетов ОС.


# **Описание процесса выполнения**

![schema](https://github.com/MsyuLuch/LinuxProfessional/blob/main/project/image/schema3.jpg)

Проект включает следующие ВМ:
- proxy
- web
- nfs
- mysql master
- mysql slave
- monitoring
- elk

На всех ВМ установлена Ubuntu 20.04, включен firewall.

В качестве web-проекта выбран Wordpress — свободно распространяемая система управления содержимым сайта с открытым исходным кодом; написана на PHP; сервер базы данных — MySQL.

<details><summary>Настройка Firewall</summary>

На каждой машине, в соответствии с её функциональностью перед началом установки необходимых пакетов, открываем порты или разрешаем работу соответствующих сервисов:
```
- name: Open Firewall for services (WEB)
  firewalld:
    service: "{{ item }}"
    permanent: yes
    state: enabled
  with_items:
    - http
    - https
```
```
- name: Open Firewall for services (MySQL, NFS)
  firewalld:
    service: "{{ item }}"
    permanent: yes
    state: enabled
  with_items:
    - mysql
    - nfs
    - mountd
    - rpc-bind  
```
```
- name: Open Firewall ports (Prometheus)
  block:
    - name: Allow Ports
      firewalld:
        port: "{{ item }}"
        permanent: true
        state: enabled
      loop: [ '9090/tcp', '9093/tcp', '9094/tcp', '9100/tcp', '9094/udp' ]
```
```
- name: Open Firewall ports (Elastic)
  block:
    - name: Allow Ports
      firewalld:
        port: "{{ item }}"
        permanent: true
        state: enabled
      loop: [ '9200/tcp','9300/tcp']
```
```
- name: Open Firewall ports (Kibana)
  block:
    - name: Allow Ports
      firewalld:
        port: "{{ item }}"
        permanent: true
        state: enabled
      loop: [ '5601/tcp']
```

```
- name: Open Firewall for port (Node exporter)
  block:
    - name: Allow Ports
      firewalld:
        port: "{{ item }}"
        permanent: true
        state: enabled
      loop: [ '9100/tcp'] 
```
</details>

<details><summary>Установка и настройка MySQL</summary>

Развернем два сервера баз данных: master и replica. Настроим между ними репликацию, в режиме Primary - Secondary.

Подключаем репозиторий `https://repo.mysql.com/` и устанавливаем MySQL 8.0:
```
- name: Install mysql repo
  yum:
    name: https://dev.mysql.com/get/mysql80-community-release-el7-5.noarch.rpm
    state: present

- name: Import a key from a url
  ansible.builtin.rpm_key:
    state: present
    key: https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

- name: Install mysql server
  yum:
    name:
      - mysql-community-server
      - MySQL-python
    state: present
  notify: Restart mysql
```

Чтобы настроить репликацию меняем id сервера и включаем режим `gtid_mode = ON` (глобальные идентификаторы транзакции).
Конфигурационные файлы обоих серверов соответственно:

master
```
[mysqld]
bind-address = {{ master_server_ip }}

gtid_mode = ON
enforce-gtid-consistency = ON

server-id = 1

log_bin = mysql-bin
log-error=/var/log/mysqld.log

replicate-do-db=wordpress
```

replica
```
[mysqld]
bind-address = {{ replica_server_ip }}

gtid_mode = ON
enforce-gtid-consistency = ON

server-id = 2
```

На реплике, внесем параметры мастера и активируем режим слэйва:
```
- name: Change to slave
  shell: mysql -u root -p'{{ mysql_root_password }}' -e 'CHANGE MASTER TO MASTER_HOST="{{ master_server_ip }}", MASTER_USER="{{ replication_user }}", MASTER_PASSWORD="{{ replication_password }}", MASTER_AUTO_POSITION=1, GET_MASTER_PUBLIC_KEY = 1;'
  
- name: Start slave
  shell: mysql -u root -p'{{ mysql_root_password }}' -e 'START SLAVE;'
```

</details>

<details><summary>Установка Wordpress</summary>

### ***Настройка Proxy***

[Как создать надежный SSL-сертификат для локальной разработки](https://medium.com/nuances-of-programming/%D0%BA%D0%B0%D0%BA-%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D0%B2%D0%B0%D1%82%D1%8C-%D0%BD%D0%B0%D0%B4%D0%B5%D0%B6%D0%BD%D1%8B%D0%B5-ssl-%D1%81%D0%B5%D1%80%D1%82%D0%B8%D1%84%D0%B8%D0%BA%D0%B0%D1%82%D1%8B-%D0%B4%D0%BB%D1%8F-%D0%BB%D0%BE%D0%BA%D0%B0%D0%BB%D1%8C%D0%BD%D0%BE%D0%B9-%D1%80%D0%B0%D0%B7%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B8-8f73f76df3d4)

Воспользуемся инструкцией и создадим самоподписанный SSL сертификат для настройки HTTPS соединения между Пользователями - Proxy - Web.

Файлы сертификата (`localhost.crt`, `localhost.key`) разместим в директориях NGINX `/etc/nginx/ssl` и Apache `/etc/httpd/ssl` соответственно.
NGINX будет слушать на порту 80 и 443, при необходимости перенаправляя весь трафик на 443 порт (https):
```
server {
        listen 80;
        return 301 https://{{ virtual_domain }}$request_uri;
        }

server {
        listen 443 ssl;

        server_name {{ virtual_domain }};

        # Указываем пути к сертификатам
        ssl_certificate /etc/nginx/ssl/localhost.crt; 
        ssl_certificate_key /etc/nginx/ssl/localhost.key;

        access_log /var/log/nginx/{{ virtual_domain }}_access.log;
        error_log /var/log/nginx/{{ virtual_domain }}_error.log;  

        location / {
                proxy_pass https://{{ web_server_ip }};
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-Proto https;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header X-Forwarded-Port 443;
                proxy_set_header Proxy "";
        }
}    
``` 

### ***Настройка Apache***

Настроим виртуальный хост Apache на работу по протоколу https:
```
<VirtualHost *:80 *:443>
   ServerAdmin webmaster@localhost
   ServerName {{ http_host }}
   DocumentRoot {{ wordpress_directory }}
   ErrorLog /var/log/httpd/{{ http_host }}_error.log
   CustomLog /var/log/httpd/{{ http_host }}_access.log combined

   <Directory {{ wordpress_directory }}>
     Options Indexes FollowSymLinks
     AllowOverride all
     Require all granted
   </Directory>
</VirtualHost>
```

### ***Настройка WordPress***
Скачиваем с официального сайта WordPress последнюю версию проекта, разархивируем файлы в рабочую директорию.
До начала работы проекта, необходимо создать базу данных `Wordpress`.

В конфигурационный файл WordPress `wp-config.php` добавим строки, принудительно переводя проект на работу по https протоколу:
```
if($_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https'){

    $_SERVER['HTTPS'] = 'on';
        $_SERVER['SERVER_PORT'] = 443;
        }

define('WP_HOME','https://{{ virtual_domain }}/');
define('WP_SITEURL','https://{{ virtual_domain }}/');
``` 

</details>

<details><summary>Установка и настройка backup</summary>

Для хранения резервных копий с сервера баз данных и веб сервера будем использовать NFS-сервер.

Установим NFS-сервер:
```
# устанавливаем NFS сервер
yum install nfs-utils -y
```

Создадим на сервере две директории, где будем хранить резервные копии веб-сайта и базы данных.
Редактируем конфигурационный файл `/etc/exports`, разрешая монтировать директории:
```
- name: Add exports param
  lineinfile:
    path: /etc/exports
    line: '{{ item }} *(rw,sync,no_root_squash,no_subtree_check,anonuid=1000,anongid=1000)'
  notify: restart nfs-server
  loop:
    - "{{ share_directory_db }}"
    - "{{ share_directory_web }}"
```  

Скрипты резервного копирования будут запускаться по cron и записывать результаты выполнения команды в log файл (дополнительно настраиваем logrotate с нужными параметрами ротации логов):
```
- name: Add backup task in cron
  lineinfile:
    path: /etc/crontab
    line: '*/20  *  *  *  *  root  /opt/backup.sh >> /var/log/my-app/backup.log 2>&1'  
```

Копии базы данных будем снимать с Master сервера. 

Mysqldump будем выполнять со следующими параметрами:
```
- single-transaction - флаг запускает транзакцию перед запуском. Вместо того, чтобы блокировать всю базу данных, это позволит mysqldump прочитать базу данных в текущем состоянии во время транзакции, создавая непротиворечивый дамп данных.
- set-gtid-purged=ON - указывает то, что используется репликация на основе глобальных идентификаторов GTID.
```

Скрипт для создания резервных копий базы данных:
```
#!/bin/bash
echo "===================================================================================================="
date
NOW=$(date +"%Y-%m-%d-%H%M")
BACKUP_DIR="{{ mount_directory_db }}"

DB_USER="root"
DB_PASS="{{ mysql_root_password }}"
DB_NAME="{{ mysql_db }}"
DB_FILE="{{ virtual_domain }}.$NOW.sql"

# Вариант создания резервной копии только одной базы данных, с указанием использования репликации
mysqldump -u$DB_USER -p$DB_PASS --single-transaction --set-gtid-purged=ON --databases $DB_NAME > $BACKUP_DIR/$DB_FILE

# Вариант создания полной резервной копии всех баз данных, с отключением опции репликации
# mysqldump -u$DB_USER -p$DB_PASS --set-gtid-purged=OFF --all-databases --triggers --routines --events > $BACKUP_DIR/$DB_FILE

if [[ $? -gt 0 ]];then
echo "ERROR: Aborted. Copying the database failed."
echo "======================================================================================================"
echo -en '\n'
exit 1
fi

echo "Copy the database successfull."
echo "======================================================================================================"
echo -en '\n'
```
Так как существует репликация и включен режим `GTID = ON` наиболее простой способ восстановить БД - развернуть Master (и Replica) с нуля с параметром `backup_flag = true`, 
описанным в общем файле `var.yml`. 

Копии frontend`а сайта снимаем с веб-сервера.
Создаем архив папки, дополнительно изменяя вложенность папок в архиве (специфика выполнения команды tar)
Скрипт backup:
```
#!/bin/bash
echo "===================================================================================================="
date
NOW=$(date +"%Y-%m-%d-%H%M")
FILE="{{ virtual_domain }}.$NOW.tar"
BACKUP_DIR="{{ mount_directory_web }}"
WWW_DIR="{{ wordpress_directory }}"

WWW_TRANSFORM='s,^{{ wordpress_directory }},wordpress,'

tar -cf $BACKUP_DIR/$FILE --absolute-names --transform $WWW_TRANSFORM $WWW_DIR > /dev/null

if [[ $? -gt 0 ]];then
echo "ERROR: Aborted. Copying the source code failed."
echo "======================================================================================================"
echo -en '\n'
exit 1
fi

echo "Copy the source code successfull."
echo "======================================================================================================"
echo -en '\n'
```

Для восстановления данных из резервной копии сайта, написана дополнительная роль `playbook-backup-web.yml`. Роль можно 
запустить с полным восстановлением веб сервера, а можно только частично, восстанавливая лишь содержимое директории, где хранится сайт.
Переменные необходимые для работы роли, хранятся в общем файле `var.yml`. 
```
- name: Restore web site
  hosts: web
  become: true
  vars_files:
    - vars.yml  
  tasks:
      - name: Recursively remove directory
        file:
          path: "{{ wordpress_directory }}"
          state: absent

      - name: Unpack files
        become: true
        unarchive:
          src: "{{ mount_directory_web }}/{{ backup_web_name }}"
          dest: "{{ wordpress_install_directory }}"
          remote_src: yes

      - name: Setting ownership
        become: true      
        file:
          path: "{{ wordpress_directory }}"
          owner: apache
          group: apache
          recurse: true
          mode: '0775' 

```
</details>


<details><summary>Установка и настройка мониторинга</summary>

Prometheus система мониторинга с открытым исходным кодом, он предоставляет десятки разных экспортеров, с помощью которых можно за считанные минуты настроить мониторинг всей инфраструктуры.

Prometheus — это база данных временных рядов. Настроенный Prometheus слушает на порту 9090:
```
# Prometheus
192.168.3.206:9090
```

В качестве экспортера выбран Node exporter, который установлен на каждом из узлов, которые необходимо мониторить. Prometheus забирает метрики у Node exporter на порту 9100.
Посмотреть метрики в текстовом виде можно:
```
# Node exporter Proxy
192.168.3.201:9100

# Node exporter Web
192.168.3.202:9100
```

Blackbox экспортер для Prometheus позволяет реализовать мониторинг внешних сервисов через HTTP, HTTPS, DNS, TCP, ICMP. 

```
# Blackbox exporter
192.168.3.206:9115
```
 
Grafana - это платформа с открытым исходным кодом для визуализации, мониторинга и анализа данных.
В данном случае она замечательно справляется с визуализацией данных, которые собрал Prometheus.

```
# Grafana
192.168.3.206:3000
```
 
</details>


<details><summary>Установка и настройка логирования</summary>


Инфраструктура ELK включает следующие компоненты:

Elasticsearch (ES) – масштабируемая утилита полнотекстового поиска и аналитики, которая позволяет быстро в режиме реального времени хранить, искать и анализировать большие объемы данных. 
Как правило, ES используется в качестве NoSQL-базы данных для приложений со сложными функциями поиска. Elasticsearch основана на библиотеке Apache Lucene, предназначенной для индексирования и поиска 
информации в любом типе документов. В масштабных Big Data системах несколько копий Elasticsearch объединяются в кластер.

Logstash — средство сбора, преобразования и сохранения в общем хранилище событий из файлов, баз данных, логов и других источников в реальном времени.  Logsatsh позволяет модифицировать полученные данные 
с помощью фильтров: разбить строку на поля, обогатить или их, агрегировать несколько строк, преобразовать их в JSON-документы и пр. Обработанные данные Logsatsh отправляет в системы-потребители. 

Kibana – визуальный инструмент для Elasticsearch, чтобы взаимодействовать с данными, которые хранятся в индексах ES. Веб-интерфейс Kibana позволяет быстро создавать и обмениваться динамическими панелями 
мониторинга, включая таблицы, графики и диаграммы, которые отображают изменения в ES-запросах в реальном времени. Примечательно, что изначально Kibana была ориентирована на работу с Logstash, а не на Elasticsearch. 
Однако, с интеграцией 3-х систем в единую ELK-платформу, Kibana стала работать непосредственно с ES.

FileBeat – агент на серверах для отправки различных типов оперативных данных в Elasticsearch.

FileBeat установлен на каждом узле сети, забирает данные из log файлов `message`, `audit`, `nginx`, `httpd`, `mysql` и отправляет эти данные на сервер логирования `elk:5044`:
```
filebeat.inputs:
- type: log
  enabled: true
  paths:
      - /var/log/httpd/*_access.log
  fields:
    type: httpd_access
  fields_under_root: true
  scan_frequency: 5s

- type: log
  enabled: true
  paths:
      - /var/log/httpd/*_error.log
  fields:
    type: httpd_error
  fields_under_root: true
  scan_frequency: 5s

- type: log
  enabled: true
  paths:
      - /var/log/nginx/*_access.log
  fields:
    type: nginx_access
  fields_under_root: true
  scan_frequency: 5s

- type: log
  enabled: true
  paths:
      - /var/log/nginx/*_error.log
  fields:
    type: nginx_error
  fields_under_root: true
  scan_frequency: 5s

- type: log
  enabled: true
  paths:
      - /var/log/messages
  fields:
    type: syslog
  fields_under_root: true
  scan_frequency: 5s

- type: log
  enabled: true
  paths:
      - /var/log/audit/audit.log
  fields:
    type: audit
  fields_under_root: true
  scan_frequency: 5s

- type: log
  enabled: true
  paths:
      - /var/log/mysqld.log
  fields:
    type: mysql
  fields_under_root: true
  scan_frequency: 5s

output.logstash:
  hosts: ["{{ elk_server_ip }}:5044"]
```

На сервере `elk` на порту 5044 слушает `Logstash`, который парсит логи с помощью `grok` фильтра и записывает в базу данных `Elasticsearch`. 

Разбиваем каждое сообщение лога NGINX, чтобы выделить поля сообщения, для создания фильтров в Kibana:
```
filter {
 if [type] == "nginx_access" {
    grok {
        match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user} \[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes} \"%{DATA:referrer}\" \"%{DATA:agent}\"" }
    }
  }
  date {
        match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
  }
}
```

Визуализацию данных можно увидеть в Kibana:
```
192.168.3.207:5601
```

</details>
