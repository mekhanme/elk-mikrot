# ELK для анализа логов Mikrotik

### ELK 7.5.0

В репозитории собрана вся конфигурация сервисов Elasticsearch, Logstash и Kibana (на данный момент версия стека 7.5.0) для обработки, анализа и визуализации логов с сетевого устрйства. 

### Honeypot

Если у вас публичный IP-адрес и более-менее умное устройство в качестве шлюза/файрволла, вы можете организовать пассивный honeypot, настроив логирование входящих запросов на "вкусные" TCP и UDP порты. В репозитории готовое решение для логирования модуля Firewall маршрутизаторов Mikrotik. 

### Grok patterns

В директории [patterns](https://github.com/mekhanme/elk-mikrot/tree/master/logstash/patterns) лежат файлы шаблонов (_grok patterns_) для Mikrotik. Пока что для задачи сбора логов модуля Firewall (в т.ч. NAT) будет использоваться шаблон _MIKROTFIREWALLNAT_. Кроме этого, там же лежит [файл](https://github.com/mekhanme/elk-mikrot/blob/master/logstash/patterns/grok-patterns) со стандартными шаблонами, которые можно использовать при написании кастомных правил.

Проверить grok паттерн можно сервисом [Grok Debugger](http://grokdebug.herokuapp.com/), либо в Kibana аналогичный функционал есть в разделе _Dev Tools_).

### Настройка Mikrotik

<details>
  <summary>Конфиг /system logging</summary>
  
```
/system logging action add remote=192.168.88.130 remote-port=5145 src-address=192.168.88.1 name=logstash target=remote
/system logging add action=logstash topics=info
```
  
</details>

<details>
  <summary>Конфиг /ip firewall</summary>
  
```
/ip firewall nat
add action=netmap chain=dstnat comment="HONEYPOT RDP" dst-port=3389 in-interface=bridge-wan log=yes log-prefix=honeypot_rdp protocol=tcp to-addresses=192.168.88.201 to-ports=3389
add action=netmap chain=dstnat comment="HONEYPOT ELASTIC" dst-port=9200 in-interface=bridge-wan log=yes log-prefix=honeypot_elastic protocol=tcp to-addresses=192.168.88.201 to-ports=9211
add action=netmap chain=dstnat comment=" HONEYPOT TELNET" dst-port=23 in-interface=bridge-wan log=yes log-prefix=honeypot_telnet protocol=tcp to-addresses=192.168.88.201 to-ports=2325
add action=netmap chain=dstnat comment="HONEYPOT DNS" dst-port=53 in-interface=bridge-wan log=yes log-prefix=honeypot_dns protocol=udp to-addresses=192.168.88.201 to-ports=9953
add action=netmap chain=dstnat comment="HONEYPOT FTP" dst-port=21 in-interface=bridge-wan log=yes log-prefix=honeypot_ftp protocol=tcp to-addresses=192.168.88.201 to-ports=9921
add action=netmap chain=dstnat comment="HONEYPOT SMTP" dst-port=25 in-interface=bridge-wan log=yes log-prefix=honeypot_smtp protocol=tcp to-addresses=192.168.88.201 to-ports=9925
add action=netmap chain=dstnat comment="HONEYPOT SMB" dst-port=445 in-interface=bridge-wan log=yes log-prefix=honeypot_smb protocol=tcp to-addresses=192.168.88.201 to-ports=9445
add action=netmap chain=dstnat comment="HONEYPOT MQTT" dst-port=1883 in-interface=bridge-wan log=yes log-prefix=honeypot_mqtt protocol=tcp to-addresses=192.168.88.201 to-ports=9883
add action=netmap chain=dstnat comment="HONEYPOT SIP" dst-port=5060 in-interface=bridge-wan log=yes log-prefix=honeypot_sip protocol=tcp to-addresses=192.168.88.201 to-ports=9060
add action=dst-nat chain=dstnat comment="HONEYPOT SSH" dst-port=22 in-interface=bridge-wan log=yes log-prefix=honeypot_ssh protocol=tcp to-addresses=192.168.88.201 to-ports=9922
```
  
</details>

Даже если у вас под рукой маршрутизатор другого вендора, нужно просто немного разобраться с форматами данных и вендоро-специфичными настройками. На отправку syslog на удалённый сервер можно настроить почти любую систему, будь то маршрутизатор, сервер или ещё какая-то security система, которая умеет логировать. Logstash слушает UDP port 5145, на который необходимо настроить отправку syslog сообщений.

### GeoLite2

Для определения географических координат по IP-адресу потребуется скачать free базу GeoLite2 от MaxMind:

<details>
  <summary>Скачивание GeoLite2</summary>
  
```
❯❯ cd elk-mikrot && mkdir logstash/geoip_db
❯❯ curl -O https://geolite.maxmind.com/download/geoip/database/GeoLite2-City-CSV.zip && unzip GeoLite2-City-CSV.zip -d logstash/geoip_db && rm -f GeoLite2-City-CSV.zip
❯❯ curl -O https://geolite.maxmind.com/download/geoip/database/GeoLite2-ASN-CSV.zip && unzip GeoLite2-ASN-CSV.zip -d logstash/geoip_db && rm -f GeoLite2-ASN-CSV.zip
```
</details>

### docker-compose

Проект запускается из _docker-compose.yml_ файла, соответственно развернуть своё подобное окружение очень просто. Достаточно иметь более-менее производительный сервер или ПК на Linux. 

Благодаря тому, что все контейнеры будут в одной сети (если явно не указывать сеть, то создаётся новый docker bridge, что подходит в этом сценарии) и в _docker-compose.yml_ у всех контейнеров прописан параметр _container_name_, между контейнерами уже будет связность через встроенный DNS докера. Вследствие чего не требуется прописывать IP-адреса в конфигах контейнеров. В конфиге Logstash прописана подсеть 192.168.88.0/24 как локальная.

В этом проекте используется docker-compose файл последней на данный момент версии 3.7, он требует docker engine версии 18.06.0+, так что не лишним обновить docker, а так же docker-compose.

<details>
  <summary>Установка docker-compose</summary>
  
```
❯❯ curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
❯❯ chmod +x /usr/local/bin/docker-compose
```
  
</details>

Так как в последних версиях docker-compose выпилили параметр _mem_limit_ и добавили _deploy_, запускающийся только в swarm mode (docker stack deploy), то запуск _docker-compose up_ конфигурации с лимитами приводит к ошибке. Так как я не использую swarm, а лимиты ресурсов иметь хочется, запускать приходится с опцией _--compatibility_, которая конвертирует лимиты из docker-compose новых версий в нон-сварм эквивалент.

Запуск всех сервисов:
```
❯❯ docker-compose --compatibility up -d
```
