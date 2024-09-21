## Задачи
Настроить nginx по заданному тз:
1. Должен работать по https c сертификатом
2. Настроить принудительное перенаправление HTTP-запросов (порт 80) на HTTPS (порт 443) для обеспечения безопасного соединения.
3. Использовать alias для создания псевдонимов путей к файлам или каталогам на сервере.
4. Настроить виртуальные хосты для обслуживания нескольких доменных имен на одном сервере.
5. Что угодно еще под требования проекта
## Установка nginx
- Сперва проверим наличие nginx - он не установлен. Установим с помощью `apt-get install nginx` 
![](img/Pasted%20image%2020240916215251.png)
- Проверим установку - всё кул! 
![](img/Pasted%20image%2020240916215641.png)
- Попробуем запустить сервис nginx - и.. окей, гуглим проблему, [тут](https://stackoverflow.com/questions/51525710/nginx-failed-to-start-a-high-performance-web-server-and-a-reverse-proxy-server) советуют проверить не запущен ли сервис apache2, который по дефолту слушает тот же 80 порт, что нужен для nginx. Действительно, таковой имеется. 
![](img/Pasted%20image%2020240916220431.png)
- Остановим apache2, запустим nginx, проверим..юху! Можем продолжать.
![](img/Pasted%20image%2020240916220952.png)
## Домены (нелокальные - неудачно)
- По тз нам нужны домены, хочется поработать не с локальными, а с реальными, поэтому ищем сайты для бесплатного выпуска доменов - находим https://nic.eu.org/. 
- По нему есть туториал в виде https://mipped.com/free/24667-poluchaem-besplatnyj-domen-euorg.html. 
- Вспомогательный сайт для создания dns-zone - https://www.cloudns.net/records/domain/8123842/.

1. Регистрируемся в обоих сервисах, указываем желаемый домен в clouds.net, запоминаем nameservers.

![](img/Pasted%20image%2020240919014258.png)
2. Идём в `nic.eu.org`, регестрируем новый домен.

![](img/Pasted%20image%2020240919014408.png)
![](img/Pasted%20image%2020240919014341.png)

3. Запрос отправлен, ждём ответа..

![](img/Pasted%20image%2020240919014239.png)

>[!IMPORTANT]
> Оказывается, по тз доменов нужно несколько, поэтому создаём поддомен `ifeelgood.idonothingso.eu.org`, надеясь, что всё в итоге будет работать, как надо..


![](img/Pasted%20image%2020240919015902.png)
> [!NOTE]
> так как данная процедура может занять >2 месяцев, то выполним работу с помощью локальных доменов и если повезёт, позже дополним работу кейсом с нелокальными доменами

## Проекты
На просторах **GitHub** был найден [репозиторий](https://github.com/bradtraversy/50projects50days) с 50 проектами, доступными к использованию и распространению, возьмём два из них.
- Создадим папку по пути `/var/www/github_profiles.com` для первого проекта и `/var/www/insect-catch-game.com` для второго.
- Создадим необходимые для проектов файлы, для их изменения применим `chmod`. Заранее создадим папки `static` для создания `alias`-ов.
![](img/Screenshot%20from%202024-09-21%2020-00-43.png)
![](img/Pasted%20image%2020240921201654.png)
## Сертификаты
Воспользуемся туториалом [отсюда](https://www.8host.com/blog/sozdanie-samopodpisannogo-ssl-sertifikata-dlya-nginx-v-ubuntu-18-04/) и создадим самоподписанный ssl сертификат для `https` соединения.
- Создадим пару ключ-сертификат с помощью утилиты `openssl`, где `-x509` указывает на то, что ключ самоподписывается, `-nodes` - на отсутствие пароля:
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout \
/etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```
- После ответа на вопросы, введём команду для поддержки PFS, которую требует `openssl`: `sudo openssl dhparam -dsaparam -out /etc/nginx/dhparam.pem 4096`
- Создаём сниппет, который укажет на местоположение ключа и сертификата: `sudo gedit /etc/nginx/snippets/self-signed.conf`
Содержимое сниппета:
```
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```
- Для настройки протокола ssl нужен еще один сниппет, создадим и его: `sudo gedit /etc/nginx/snippets/ssl-params.conf`
Содержимое сниппета:
```
ssl_protocols TLSv1.2;   
ssl_prefer_server_ciphers on;   
ssl_dhparam /etc/nginx/dhparam.pem;   
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;   
ssl_ecdh_curve secp384r1; # Requires nginx >= 1.1.0   
ssl_session_timeout  10m;   
ssl_session_cache shared:SSL:10m;   
ssl_session_tickets off; # Requires nginx >= 1.5.9   
ssl_stapling off; # Requires nginx >= 1.3.7   
ssl_stapling_verify off; # Requires nginx => 1.3.7   
resolver 8.8.8.8 8.8.4.4 valid=300s;   
resolver_timeout 5s;   
# Disable strict transport security for now. You can uncomment the following   
# line if you understand the implications.   
# add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"; add_header X-Frame-Options DENY;   
add_header X-Content-Type-Options nosniff;   
add_header X-XSS-Protection "1; mode=block";
```
## Настройка nginx
- Создадим конфиги для nginx под каждый проект с помощью дефолтного конфига nginx -
```
sudo cp /etc/nginx/sites-available/default \
/etc/nginx/conf.d/<github_profiles or insect-catch-game>.conf
```
### Принудительное перенаправление трафика с http на https
Чтобы при попытке подключиться по http производился редирект на https, добавим в конфиги следующее:
```
server {
	listen 80;
	listen [::]:80;
	server_name github_profiles.com;
	return 301 https://$server_name$request_uri;

}

server {

    listen 443 ssl;
	listen [::]:443 ssl;
	include snippets/self-signed.conf;
	include snippets/ssl-params.conf;
    server_name github_profiles.com;
    index index.html;
    root /var/www/github_profiles.com;
}


```

![](img/Pasted%20image%2020240922002449.png)
### Ничего не получилос..
Было удачно забыто, что используется ***ЛОКАЛЬНЫЙ*** хост..
Исправим ошибку добавлением строчки с ip хоста и локальным доменом в `/etc/hosts`: `127.0.0.1       github_profiles.com`.
Показалось, что ничего не поменялось, но чистка кеша дала свои плоды:
![](img/Pasted%20image%2020240922004944.png)
***WIN WIN WIN !!!***
Соединение незащищено, приложение не работает, но оно хотя бы доступно)
*Upd: приложение заработало после небольших корректировок в `index.html`:
![](img/Pasted%20image%2020240922024705.png)
### Добавление сертификата в браузер