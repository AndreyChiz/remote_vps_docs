

# Структура каталогов

- /home/www/public_sites - каталог для сайта 
- /home/www/public_sites/iot - каталог для сайта 
- /home/www/docker -   приложения окружения
- /home/www/log - логи приложений
- /home/www/docs - документация
- /home/www/src - исходники

## log

### Создание пользователя www с домашней директорией и shell
sudo adduser -h /home/www -s /bin/bash www  # пользователь сервер, shell bash, домашняя директория /home/www

### Настройка прав на домашнюю директорию
sudo chown -R www:www /home/www            # владелец и группа www
sudo chmod 700 /home/www                    # только владелец имеет rwx

### Создание структуры каталогов
- sudo mkdir -p /home/www/dev                                # каталог для разработки

sudo mkdir -p /home/www/public_sites/blog                  # каталог для сайта blog
sudo mkdir -p /home/www/public_sites/iot                   # каталог для сайта iot
sudo mkdir -p /home/www/docker                             # конфиги и docker-compose
sudo mkdir -p /home/www/log                               # логи приложений

### Присваиваем владельца www
sudo chown -R www:www /home/www/dev
sudo chown -R www:www /home/www/public_sites
sudo chown -R www:www /home/www/docker
sudo chown -R www:www /home/www/logs

### Настройка прав доступа
sudo chmod 700 /home/www/dev
sudo chmod 755 /home/www/public_sites       # публичные сайты доступны для чтения
sudo chmod 700 /home/www/docker
sudo chmod 700 /home/www/logs

### Проверка работы под пользователем www
su www
cd /home/www/dev
mkdir test_dir                               # должно работать

### Добавление пользователя achi в группу www
sudo usermod -aG www achi                    # achi получает доступ через группу www

### Настройка прав группы на домашнюю директорию
sudo chown -R www:www /home/www
sudo chmod -R 770 /home/www                  # владелец и группа имеют полный доступ

### Альтернатива через ACL, если группа не даёт доступ
sudo setfacl -R -m u:achi:rwx /home/www     # доступ к существующим файлам
sudo setfacl -R -d -m u:achi:rwx /home/www # доступ к новым файлам и каталогам

### Проверка доступа пользователя achi
su achi
cd /home/www
touch test_file                             