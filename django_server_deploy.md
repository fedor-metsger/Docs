
### Создание пользователя
```
useradd
usermod username -aG sudo
```

### Скачиваем проект
```
git clone <github_url>
```

### Устанавливаем пакеты
```
sudo apt install python3-venv python3-pip postgresql nginx expect python-is-python3 libpq-dev
```

### Создаём виртуальное окружение, устанавливаем модули
```
python -m venv venv
. venv/bin/activate
pip install -r requirements.txt
pip freeze
```

### Создаём БД
```
sudo su postgres
psql
ALTERUSER postgres WITH PASSWORD '123456';
CREATE DATABASE db_name;
\q
exit
```

### Правим `settings.py`
```
DEBUG=False
ALLOWED_HOSTS=
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'db_name',
        'HOST': '127.0.0.1',
        'PORT': '5432',
        "USER": "postgres",
        "PASSWORD": "postgres"
    }
}
```
### Прогоняем миграции и запускаем сервер
```
./manage.py migrate
./manage.py runserver 0.0.0.0:8000
```

### Устанавливаем `gunicorn` в виртуальном окружении
```
. venv/bin/activate
pip install gunicorn
cat > /etc/systemd/system/gunicorn.service <<EOF
[Unit]
Description=Service for gunicorn
After=network.target

[Service]
User=<user_name>
WorkingDirectory=/home/<user_name>/<project_dir>
ExecStart=/home/<user_name>/<project_dir>/venv/bin/gunicorn --workers 3 --bind unix:/home/<user_name>/<project_dir>/project.sock config.wsgi:application

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl start gunicorn.service
sudo systemctl enable gunicorn.service
```

### Настраиваем `nginx`
```
cat > /etc/nginx/sites-available/<project_name> <<EOF
server {
  listen 80;
  server_name <IP_ADDRESS>;
  location /static/ {
    root /home/<user_name>/<project_name>;
  }
  location / {
    include proxy_params;
    proxy_pass http://unix:/home/<user_name>/<project_dir>/project.sock;
  }
}
EOF
ln -s /etc/nginx/sites-available/<project_name> /etc/nginx/sites-enabled
systemctl start nginx
```
### Собираем статику в кучу
```
. venv/bin/activate
./manage.py collectstatic
```
