# Внутренний сервер

NAT (chiz - xiaomi-redmi-note4x CPU8x2GРС arm64 RAM=3G EMMS=24Gb)

### Алиасы и скрипты

- proxy_log - лог nginx
- api_log - лог gunicorn


```sh
alias wol_home_laptop_17='wakeonlan DC:0E:A1:B7:21:79'
alias wol_home_sharaban='wakeonlan 00:1D:60:17:5C:DC'
alias batt='cat /sys/class/power_supply/qcom-battery/capacity && cat /sys/class/power_supply/qcom-battery/status'
alias wifi_scan='nmcli device wifi list'
alias lcd_off='echo 1 | sudo tee /sys/class/backlight/backlight/bl_power'
alias lcd_on='echo 0 | sudo tee /sys/class/backlight/backlight/bl_power'
alias to_screen='sudo openvt -f -c 1 --'
alias cpu_max='echo "performance" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor'
alias cpu_norm='echo "schedutil" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor'
alias cpu_min='echo "powersave" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor'
alias phone_mode='sudo systemctl set-default graphical.target'
alias console_mode='sudo systemctl set-default multi-user.target'
alias bigfont='sudo setfont /usr/share/consolefonts/ter-p24b.psf.gz'
alias api_log='llog /home/www/log/chiz.work.gd/gunicorn_back/chiz.work.gb_access.log'
alias proxy_log='llog /home/www/log/chiz.work.gd/nginx_front/chiz_frontend_access.log'




#Сканирование wifi
wifi() {
    # Сначала запустим wavemon для сканирования сетей
    echo "Запуск wavemon для сканирования сетей..."
    echo "Используйте F3 для просмотра списка сетей"
    echo "Запомните имя сети (SSID) и нажмите q для выхода"
    sudo wavemon
    
    # После выхода из wavemon запросим имя сети
    if [ "$1" ]; then
        SSID="$1"
    else
        echo -n "Введите имя сети (SSID): "
        read SSID
    fi
    
    # Подключение с запросом пароля
    sudo nmcli device wifi connect "$SSID" --ask
}

# To customize prompt, run `p10k configure` or edit ~/.p10k.zsh.
[[ ! -f ~/.p10k.zsh ]] || source ~/.p10k.zsh


#Скрипт для настройки яркости
backlight() {
  if [[ -z $1 ]]; then
    echo "Использование: backlight <проценты от 0 до 100>"
    return 1
  fi
  if ! [[ $1 =~ ^[0-9]+$ ]]; then
    echo "Ошибка: аргумент должен быть числом"
    return 1
  fi
  if (( $1 < 0 || $1 > 100 )); then
    echo "Ошибка: число должно быть от 0 до 100"
    return 1
  fi

  max=$(cat /sys/class/backlight/backlight/max_brightness)
  # Вычисляем значение яркости по процентам
  value=$(( max * $1 / 100 ))

  echo $value | sudo tee /sys/class/backlight/backlight/brightness
}

# llog: подсвеченные логи с помощью ccze
llog() {
  if [ -z "$1" ]; then
    echo "Использование: llog <имя_файла>"
    return 1
  fi

  if [ ! -f "$1" ]; then
    echo "Файл не найден: $1"
    return 1
  fi

  tail -f "$1" | ccze -A
}

```


### Systemd

## Сетевые службы

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
## Локальные службы

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