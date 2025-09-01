# Юниты


## Сервер за NAT (chiz - xiaomi-redmi-note4x arm64)

### SSH тунель - обратный прокси
/etc/syst/system/reverse-tunnel.service 

```conf                          
[Unit]
Description=Persistent reverse SSH tunnel to VPS
After=network.target

[Service]
User=achi
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 0 -N \
  -o "ServerAliveInterval=30" \
  -o "ServerAliveCountMax=1000" \
  -R 8080:localhost:80 \
  -R 8443:localhost:443 \
  -R 2228:localhost:228 \
  -R 8000:localhost:8001 \
  achi@91.132.162.205
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### nginx
/home/www/docker/nginx/conf/chiz.work.gb.conf


```conf
# Расширенный формат логов
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time "$host" "$scheme" "$http_cookie"';

server {
    listen 80;
    server_name chiz.work.gd;

    # Корень фронтенда
    root /usr/share/nginx/html/chiz.work.gd-frontend;
    index index.html;

    # Отдача статики
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Прокси для API
    location /api/ {
        proxy_pass http://127.0.0.1:8001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Кеширование статических файлов
    location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|svg|json)$ {
        expires 1M;
        access_log off;
        add_header Cache-Control "public";
    }

    # Логи
    access_log /var/log/nginx/chiz_frontend_access.log detailed;
    error_log /var/log/nginx/chiz_frontend_error.log debug;
}
```


### Настройка отображения адресов на экране

/etc/syst/system/tty-setup.service

```conf
[Unit]
Description=Setup console font and rotate tty
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/tty-setup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target

```
/usr/local/bin/tty-setup.sh
```sh
#!/bin/sh
setfont /usr/share/consolefonts/sun12x22.psfu.gz
echo 1 > /sys/class/graphics/fbcon/rotate

```



/etc/syst/system/network-info.service

```conf
[Unit]
Description=Display network information at boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/update-network-info.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
/usr/local/bin/update-network-info.sh
```sh
#!/bin/sh
# Функция для обновления информации
update_info() {
    # Попытка дождаться подключения к сети (до 30 секунд)
    for i in $(seq 1 30); do
        WIFI_INTERFACE=$(ip -o -4 route show to default | awk '{print $5}' 2>/dev/null)
        if [ -n "$WIFI_INTERFACE" ]; then
            break
        fi
        sleep 1
    done

    # Если интерфейс все еще не найден, попробуем найти его напрямую
    if [ -z "$WIFI_INTERFACE" ]; then
        WIFI_INTERFACE=$(ip link | grep -E 'wlan|wlp' | cut -d: -f2 | awk '{print $1}' | head -n1)
    fi

    # Получаем SSID и IP
    if [ -n "$WIFI_INTERFACE" ]; then
        SSID=$(iwgetid -r 2>/dev/null || echo "Not connected")
        IP_ADDRESS=$(ip -4 addr show dev $WIFI_INTERFACE 2>/dev/null | grep -oP '(?<=inet\s)\d+(\.\d+){3}' || echo "No IP")
    else
        SSID="Not connected"
        IP_ADDRESS="No IP"
    fi

    # Обновляем файл issue
    cat > /etc/issue <<EOF
Welcome to Alpine Linux!
-------------------------
Network: $SSID
IP: $IP_ADDRESS
-------------------------

EOF
}

# Первое обновление
update_info

# Если передан параметр --delayed, выполняем обновление через минуту
if [ "$1" = "--delayed" ]; then
    sleep 60
    update_info
fi

```

/etc/syst/system/show-ip.service
```conf
[Unit]
Description=Show IP information at boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/show-ip.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

/usr/local/bin/show-ip.sh
```sh
#!/bin/bash
# Ждем немного, чтобы сеть успела подключиться
sleep 10

# Получаем информацию о сети
IP_INFO=$(ip a)

# Создаем файл issue
cat > /etc/issue <<EOF
Welcome to Alpine Linux!
-------------------------
$IP_INFO
-------------------------

EOF


```

## Vps 

### Конфиг UFW 
/etc/ufw/before.rules
```yaml
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

# Don't delete these required lines, otherwise there will be errors
*filter
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]
# End required lines


# allow all on loopback
-A ufw-before-input -i lo -j ACCEPT
-A ufw-before-output -o lo -j ACCEPT

# quickly process packets for which we already have a connection
-A ufw-before-input -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A ufw-before-output -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A ufw-before-forward -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# ВАЖНО!!!
# drop INVALID packets (logs these in loglevel medium and higher)
#-A ufw-before-input -m conntrack --ctstate INVALID -j ufw-logging-deny
#-A ufw-before-input -m conntrack --ctstate INVALID -j DROP

# ok icmp codes for INPUT
-A ufw-before-input -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-input -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-input -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-input -p icmp --icmp-type echo-request -j ACCEPT

# ok icmp code for FORWARD
-A ufw-before-forward -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-forward -p icmp --icmp-type echo-request -j ACCEPT

# allow dhcp client to work
-A ufw-before-input -p udp --sport 67 --dport 68 -j ACCEPT

#
# ufw-not-local
#
-A ufw-before-input -j ufw-not-local

# if LOCAL, RETURN
-A ufw-not-local -m addrtype --dst-type LOCAL -j RETURN

# if MULTICAST, RETURN
-A ufw-not-local -m addrtype --dst-type MULTICAST -j RETURN

# if BROADCAST, RETURN
-A ufw-not-local -m addrtype --dst-type BROADCAST -j RETURN

# all other non-local packets are dropped
-A ufw-not-local -m limit --limit 3/min --limit-burst 10 -j ufw-logging-deny
-A ufw-not-local -j DROP

# allow MULTICAST mDNS for service discovery (be sure the MULTICAST line above
# is uncommented)
-A ufw-before-input -p udp -d 224.0.0.251 --dport 5353 -j ACCEPT

# allow MULTICAST UPnP for service discovery (be sure the MULTICAST line above
# is uncommented)
-A ufw-before-input -p udp -d 239.255.255.250 --dport 1900 -j ACCEPT

# don't delete the 'COMMIT' line or these rules won't be processed
COMMIT


```

### Nginx конфиг

/etc/nginx/sites-available/reverse-proxy.conf
```sh
  GNU nano 4.8                                          /etc/nginx/sites-available/reverse-proxy.conf                                                     
    server_name chiz.work.gd;

    ssl_certificate /etc/letsencrypt/live/chiz.work.gd/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chiz.work.gd/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }


    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/chiz.work.gd.access.log;
    error_log  /var/log/nginx/chiz.work.gd.error.log;
}

server {
    listen 80;
    server_name chiz.work.gd;

    return 301 https://$host$request_uri;
}

```