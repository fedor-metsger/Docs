
### Cоздаём переменные окружения

В **Github Secrets** создаём переменные окружения:
```
ALLOWED_HOSTS=
SECRET_KEY=
DB_ENGINE=
DB_NAME=
DB_USER=
DB_PASSWORD=
DB_HOST=
DB_PORT=
SSH_HOST=
SSH_USER=
SSH_PASSWORD=
```

### Создаём в проекте файл конфигурации
```
mkdir -p .github/workflows
cat > .github/workflows/cicd.yml <<EOF
name: Django project testing and deployment

on:
  push:
    branches:
      - ci_cd

jobs:
  django-tests:
    runs-on: ubuntu-22.04
    env:
      SECRET_KEY: ${{ secrets.SECRET_KEY }}
      DEBUG: "0"
      ALLOWED_HOSTS: ${{ secrets.ALLOWED_HOSTS }}
      DB_ENGINE: ${{ secrets.DB_ENGINE }}
      DB_NAME: ${{ secrets.DB_NAME }}
      DB_HOST: ${{ secrets.DB_HOST }}
      DB_PORT: ${{ secrets.DB_PORT }}
      DB_USER: ${{ secrets.DB_USER }}
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    services:
      postgres_main:
        image: postgres:12
        env:
          POSTGRES_DB: ${{ env.DB_NAME }}
          POSTGRES_USER: ${{ env.DB_USER }}
          POSTGRES_PASSWORD: ${{ env.DB_PASSWORD }}
        ports:
          - 5432:5432
        options:
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5
    steps:
      - name: Проверка репозитория на наличие изменений
        uses: actions/checkout@v3
      - name: Установка python и окружения
        uses: actions/setup-python@v4
        with: 
          python-version: '3.10'
      - name: Установка зависимостей
        run: pip install -r requirements.txt
      - name: linter
        run: flake8 <project_dir> --exclude <project_dir>/migtrations
      - name: Тестирование
        run: ./manage.py test
        env:
          SECRET_KEY: ${{ env.SECRET_KEY }}
          DEBUG: ${{ env.DEBUG }}
          ALLOWED_HOSTS: ${{ env.ALLOWED_HOSTS }}
          DB_ENGINE: ${{ env.DB_ENGINE }}
          DB_NAME: ${{ env.DB_NAME }}
          DB_HOST: ${{ env.DB_HOST }}
          DB_PORT: ${{ env.DB_PORT }}
          DB_USER: ${{ env.DB_USER }}
          DB_PASSWORD: ${{ env.DB_PASSWORD }}
      - name: Деплой
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          password: ${{ secrets.SSH_PASSWORD }}
          script: expect /home/<user_name>/<project_name>/deploy.exp
EOF
```

### Создаём `bash` скрипт для деплоя
```
cd <project_dir>
cat > deploy.sh <<EOF
#!/bin/bash
cd /home/<user_name>/<project_name>
git pull origin ci_cd
. venv/bin/activate
pip install -r requirements.txt
./manage.py migrate
sudo systemctl restart gunicorn
EOF
cat > deploy.exp <<EOF
#!/bin/bash/expect
spawn /home/<user_name>/<project_dir>/deploy.sh
expect "password"
send --"123456\r"
expect eof
EOF
sudo chmod +x deploy.sh deploy.exp
```
