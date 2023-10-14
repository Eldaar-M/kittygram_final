# Описание проекта Kittygram

Kittygram — сервис для публикации данных о котиках. В публикациях можно выкладывать фотографии, выбирать цвет и возраст котика.
Публикации также можно редактировать и удалять.

# Технологии

- Python 3.x
- node.js 9.x.x
- frontend: React
- backend: Django
- nginx
- gunicorn
- Certbot

# Установка
## Клонирование проекта с GitHub на сервер:
```
git clone git@github.com:Eldaar-M/infra_sprint1.git
```

## Настройка бэкенд-приложения

1. Установите зависимости из файла requirements.txt:

Перейдите в директорию бэкенд-приложения проекта.
```
cd infra_sprint1/backend/
```

Создайте виртуальное окружение.
```
python3 -m venv venv
```

Активируйте виртуальное окружение.
```
source venv/bin/activate
```

Установите зависимости.
```
pip install -r requirements.txt
```
<br>2. Выполните миграции и создайте суперюзера из директории с файлом manage.py:

Примените миграции.
```
python3 manage.py migrate
```

Создайте суперпользователя.
```
python3 manage.py createsuperuser
```
3. Добавьте в список ALLOWED_HOSTS внешний IP сервера, 127.0.0.1, localhost и домен:
Вместо xxx.xxx.xxx.xxx укажите IP сервера, а вместо <ваш_домен> – доменное имя
```
ALLOWED_HOSTS = ['xxx.xxx.xxx.xxx', '127.0.0.1', 'localhost', 'ваш_домен']
```
4. Откройте файл settings.py и отключите режим дебага для бэкенд-приложения:
```
from pathlib import Path
...
# Вот тут нужно поменять значение с True на False.
DEBUG = False
...
```
5. Соберите статику бэкенд-приложения:
```
python3 manage.py collectstatic
```
6. Скопируйте директорию static_backend/ в директорию /var/www/kittygram/:
```
sudo cp -r infra_sprint1/backend/static_backend /var/www/kittygram
```
## Настройка фронтенд-приложения
1. Находясь на сервере, из любой директории выполните команду (скопируйте её целиком и вставьте в терминал):
```
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash - &&\
sudo apt-get install -y nodejs 
```

Далее убедитесь, что пакетный менеджер npm тоже установился. Выполните команду:
```
npm -v 
```
2. Находясь в директории с фронтенд-приложением, установите зависимости для него:
```
npm i
```
3. Из директории с фронтенд-приложением выполните команду:
```
npm run build
```
4. Скопируйте статику фронтенд-приложения в директорию по умолчанию:
```
sudo cp -r <имя_пользователя>/infra_sprint1/frontend/build/. /var/www/kittygram/
```
## Установка и настройка WSGI-сервера Gunicorn
1. Подключитесь к удалённому серверу, активируйте виртуальное окружение
бэкенд-приложения и установите пакет gunicorn:
```
pip install gunicorn==20.1.0
```
2. Создайте файл конфигурации юнита systemd для Gunicorn в директории
/etc/systemd/system/. Назовите его по шаблону gunicorn_название_проекта.service:
```
sudo nano /etc/systemd/system/gunicorn_kittygram.service
```
3. Подставьте в код из листинга свои данные, добавьте этот код без комментариев в файл
конфигурации Gunicorn и сохраните изменения:
```
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=имя_пользователя_в_системе
WorkingDirectory=/home/имя_пользователя/infra_sprint1/backend/
ExecStart=/.../venv/bin/gunicorn --bind 0.0.0.0:8000 backend.wsgi:application

[Install]
WantedBy=multi-user.target
```
Команда sudo systemctl с параметрами start, stop или restart запустит, остановит
или перезапустит Gunicorn.
```
sudo systemctl start gunicorn_kittygram
```
Чтобы systemd следил за работой демона Gunicorn, запускал его при старте системы
и при необходимости перезапускал, используйте команду:
```
sudo systemctl enable gunicorn_название_kittygram
```
## Установка и настройка веб- и прокси-сервера Nginx
1. Если Nginx ещё не установлен на удалённый сервер, установите его:
```
sudo apt install nginx -y
```
2. Запустите Nginx командой:
```
sudo systemctl start nginx
```
3. Обновите настройки Nginx. Для этого откройте файл конфигурации веб-сервера…
```
sudo nano /etc/nginx/sites-enabled/default
```
…очистите содержимое файла и запишите новые настройки:
```
server { 
    server_name ваш_домен; 
 
    location /media/ { 
        root /var/www/kittygram; 
    } 
 
    location /api/ { 
        proxy_pass http://127.0.0.1:8080; 
	client_max_body_size 20M; 
    } 
 
    location /admin/ { 
        proxy_pass http://127.0.0.1:8080; 
	client_max_body_size 20M; 
    } 
 
    location / { 
        root   /var/www/kittygram; 
        index  index.html index.htm; 
        try_files $uri /index.html; 
    }
```
4. Сохраните изменения в файле, закройте его и проверьте на корректность:
```
sudo nano /etc/nginx/sites-enabled/default
```
5. Перезагрузите конфигурацию Nginx:
```
sudo systemctl reload nginx
```

## Настройка файрвола ufw
1. Активируйте разрешение принимать запросы.
```
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
```
2. Включите файрвол:
```
sudo ufw enable
```
## Получение и настройка SSL-сертификата
1. Находясь на сервере, установите certbot, если он ещё не установлен:
```
# Установка пакетного менеджера snap.
sudo apt install snapd
# Установка и обновление зависимостей для пакетного менеджера snap.
sudo snap install core; sudo snap refresh core
# Установка пакета certbot.
sudo snap install --classic certbot
# Обеспечение доступа к пакету для пользователя с правами администратора.
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
2. Запустите certbot и получите SSL-сертификат. Для этого выполните команду:
```
sudo certbot --nginx
```
Далее система попросит вас указать электронную почту и ответить на несколько вопросов. Сделайте это.
Следующим шагом укажите имена, для которых вы хотели бы активировать HTTPS. Введите номер нужного доменного имени и нажмите Enter

3. Проверьте конфигурацию Nginx, и если всё в порядке, перезагрузите её.
##
Создайте `.env` файл в директории `/backend/`, в котором должна содержаться следующая переменная: 
>SECRET_KEY= # секретный ключ\ 
 
## Автор 
- DevOps: Эльдар Магомедов

#  Как работать с репозиторием финального задания

## Что нужно сделать

Настроить запуск проекта Kittygram в контейнерах и CI/CD с помощью GitHub Actions

## Как проверить работу с помощью автотестов

В корне репозитория создайте файл tests.yml со следующим содержимым:
```yaml
repo_owner: ваш_логин_на_гитхабе
kittygram_domain: полная ссылка (https://доменное_имя) на ваш проект Kittygram
taski_domain: полная ссылка (https://доменное_имя) на ваш проект Taski
dockerhub_username: ваш_логин_на_докерхабе
```

Скопируйте содержимое файла `.github/workflows/main.yml` в файл `kittygram_workflow.yml` в корневой директории проекта.

Для локального запуска тестов создайте виртуальное окружение, установите в него зависимости из backend/requirements.txt и запустите в корневой директории проекта `pytest`.

## Чек-лист для проверки перед отправкой задания

- Проект Taski доступен по доменному имени, указанному в `tests.yml`.
- Проект Kittygram доступен по доменному имени, указанному в `tests.yml`.
- Пуш в ветку main запускает тестирование и деплой Kittygram, а после успешного деплоя вам приходит сообщение в телеграм.
- В корне проекта есть файл `kittygram_workflow.yml`.
