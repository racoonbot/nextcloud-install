`sudo apt-get update -y`
`sudo apt-get update -y`

# Установка LAMP (Installing LAMP Server) 

`sudo apt-get install apache2 mariadb-server -y`

я буду использовать версию php 7.3 . это будет упоминаться в некоторых командах. у вас можеть быть другая !!!

`sudo apt-get install php7.3-xml php7.3 php7.3-cgi php7.3-cli php7.3-gd php7.3-curl php7.3-zip php7.3-mysql php7.3-mbstring wget unzip -y
`
`sudo apt-get install php7.3-xml`

`sudo apt-get install libapache2-mod-php7.3
 libapache2-mod-php`
 
Запускаем Апач и ставим его в автозагрузку sudo systemctl start apache2 sudo systemctl enable apache2 на этом этапе по адресу должна быть видна заглушка апача тоже самое с БД
 `sudo systemctl start mariadb.service `
 `sudo systemctl enable mariadb.service`

Настройка MARIADB (Configure MariaDB) sudo mysql_secure_installation пароль необязательно отвечаем на вопросы по порядку входим в MARIADB (log in to MariaDB) 

`sudo mysql -u root -p` (если выше пароль для рута то используем его)

(!!!в дальнейших командах символ ; НЕ ЗАБЫВАЕМ!!!))) 



# Создаём БД


`MariaDB [(none)]>CREATE DATABASE nextclouddb; 
создаём админа БД
MariaDB [(none)]>CREATE USER 'имя'@'localhost' IDENTIFIED BY 'пароль'; 
даём права
MariaDB [(none)]>GRANT ALL PRIVILEGES ON nextclouddb.* TO '(ИМ ЯПОЛЬЗОВАТЕЛЯ БД)'@'localhost';
MariaDB [(none)]>FLUSH PRIVILEGES;  
УХодим
MariaDB [(none)]>\q`

# Устанавливаем сам NextCloud (Install NextCloud)
 (УКАЗЫВАЙТЕ ВЕРСИЮ ВЕРНО тут 13.0.2)
  качаем 
   `wgethttps://download.nextcloud.com/server/releases/nextcloud-13.0.2.zip`
  
  распаковываем
   `unzip nextcloud-13.0.2.zip`
   перемещаем
   `sudo mv nextcloud /var/www/html/`
   
   права
`sudo chown -R www-data:www-data /var/www/html/nextcloud`

# Создаём  virtual host 

`sudo nano /etc/apache2/sites-available/nextcloud.conf `
в который вставляем текст


    <VirtualHost *:80>
    ServerAdmin admin@example.com
    DocumentRoot "/var/www/html/nextcloud"
    ServerName 192.168.0.187
    <Directory "/var/www/html/nextcloud/">
    Options MultiViews FollowSymlinks
    
    AllowOverride All
    Order allow,deny
    Allow from all
    </Directory>
    TransferLog /var/log/apache2/nextcloud_access.log
    ErrorLog /var/log/apache2/nextcloud_error.log
    </VirtualHost>
    
Сохранили. Закрыли. (safe/close)


`sudo a2ensite nextcloud`

`sudo systemctl reload apache2`

# Брендмауер если надо
 `sudo apt-get install ufw -y sudo ufw allow 22`  (НЕ ЗАБЫВЕМ ДОБАВИТЬ ПОРТ SSH если используете ) 
 
 `sudo ufw allow 80 sudo ufw enable`

# Запускаем Вэб интерфейс (start Web Interface)
 1. Открываем в браузере http://192.168.0.187 (или чо там у вас за локальный адрес)
  2. входим по имени и паролю пользователь 
  3. указываем имя и пароль базы данных
 
 В терминале включаем доп. Модули апача

`sudo a2enmod rewrite`

`sudo systemctl reload apache2`

`sudo a2enmod headers env dir mime`

`sudo systemctl reload apache2`

# Конфиг Apache2
 `sudo nano /etc/php/7.3/apache2/php.ini`
 
   находим строки и меняем (_для поиска в nano в открытом файле нажмиме ctrl+w)_



В файле много текста. Листайте, пока не найдёте раздел, посвящённый opcache, затем вставьте/измените туда/там следующие параметры

`opcache.enable=1
opcache.enable_cli=1
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1`

снимаем ограничения на загрузку

`upload_max_filesize=100M
post_max_size=100M`

# включаем ssl

создаем в папке /etc/apache2/sites-available/ еще один файл хоста 

`sudo nano /etc/apache2/sites-available/nextcloud-le-ssl.conf`

`<VirtualHost *:443>

ServerAdmin admin@example.com
DocumentRoot "/var/www/html/nextcloud"
ServerName example.com
<Directory "/var/www/html/nextcloud/">
Options MultiViews FollowSymlinks


AllowOverride All
Order allow,deny
Allow from all
</Directory>
TransferLog /var/log/apache2/nextcloud_access.log
ErrorLog /var/log/apache2/nextcloud_error.log`

`SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem`
`SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem`
`Include /etc/letsencrypt/options-ssl-apache.conf`

</VirtualHost>
ДАЛЕЕ ВКЛЮЧАЕМ ССЛ версию хоста

sudo a2enmod ssl

sudo systemctl reload apache2

sudo a2ensite nextcloud-le-ssl.conf

sudo systemctl reload apache2

Редактируем .htaccess** ** sudo nano /var/www/html/nextcloud/.htaccess Сразу после строчки

<IfModule mod_headers.c>
добавьте

Header always set Strict-Transport-Security "max-age=15768000; includeSubDomains; preload"


# Настройка Memcached 

Устанавливаем Memcached sudo apt install php-memcached memcached

Проверим запустился ли демон

netstat -tap | grep memcached
tcp 0 0 localhost:11211 *:* LISTEN 13300/memcached

или так #ps ax | grep memcached

13300 ?        Ssl    0:00 /usr/bin/memcached -m 64 -p 11211 -u memcache -l 127.0.0.1
13424 pts/3    S+     0:00 grep --color=auto memcached
Добавляем настройки для работы с Memcached в конфигурационный файл NextCloud sudo nano /var/www/html/nextcloud/config/config.php

следующую строки: (я просто вставил с заменой всего и проканало)

 'memcache.distributed' => '\OC\Memcache\Memcached',
  'memcache.local' => '\OC\Memcache\Memcached',
  'memcached_servers' => array(
      array('localhost', 11211),
   ),
  'memcached_options' => array(
        \Memcached::OPT_CONNECT_TIMEOUT => 50,
        \Memcached::OPT_RETRY_TIMEOUT =>   50,
        \Memcached::OPT_SEND_TIMEOUT =>    50,
        \Memcached::OPT_RECV_TIMEOUT =>    50,
        \Memcached::OPT_POLL_TIMEOUT =>    50,
        // Enable compression
        \Memcached::OPT_COMPRESSION =>          true,
        // Turn on consistent hashing
        \Memcached::OPT_LIBKETAMA_COMPATIBLE => true,
        // Enable Binary Protocol
        \Memcached::OPT_BINARY_PROTOCOL =>      true,
   ),
Перезагрузим сервис Apache sudo service apache2 restart

далее сертификаты ssl

certbot

sudo apt install snapd

sudo snap install core; sudo snap refresh core

sudo snap install --classic certbot

sudo ln -s /snap/bin/certbot /usr/bin/certbot

sudo certbot --apache **

на вопросики отвечаем

там нужно будет выбрать ваш домен вида example.com

**включаем перенаправление **(выбираем 2 в ответе на redict)

https://t.me/runextcloud

https://matrix.to/#/!VlQUCPyhZogQzOnExW:matrix.org?via=matrix.org&via=integrations.ems.host