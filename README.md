# Лабораторная работа по работе с docker
---
# Домашнее задание

Для начала рассмотрим все файлы, которые содержатся в данном проекте:
## `docker-compose.yml`
```sh
services:
  app:
    build: .
    container_name: lab_docker
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy
    env_file:
      - .env

  db:
    image: mysql:8.0
    container_name: mysql_db
    restart: always
    env_file:
      - .env
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:
```

---
## `Dockerfile`
```sh
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y build-essential

COPY app/requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app/ .

CMD ["python", "app.py"]
```
---

### `db/init.sql`
```sh
SET NAMES utf8mb4;

CREATE TABLE items (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

INSERT INTO items (name) VALUES ('Пример 1'), ('Пример 2');
```
Данный файл создаёт таблицу items с полями id и name, принудительно задав кодировку utf8mb4 для хранения данных. 

<details> <summary>Вывод прошлой версии программы </summary>

Список из Базы Данных <br>
ÐŸÑ€Ð¸Ð¼ÐµÑ€ 1  <br>
ÐŸÑ€Ð¸Ð¼ÐµÑ€ 2  <br>

</details>

---

### `app/app.py`
```sh
from flask import Flask, render_template
from models import ItemModel

app = Flask(__name__)
model = ItemModel()

@app.route('/')
def index():
    # Контроллер запрашивает данные у модели
    items = model.get_all_items()
    # И передает их в представление (шаблон)
    return render_template('index.html', items=items)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

### `app/models.py`
```sh
import os
import mysql.connector

class ItemModel:
    def __init__(self):
        self.config = {
            'host': os.getenv('DB_HOST', 'db'),
            'port': 3306,
            'user': os.getenv('DB_USER', 'user'),
            'password': os.getenv('DB_PASSWORD', 'userpass'),
            'database': os.getenv('DB_NAME', 'mydb'),
            'charset': 'utf8mb4',          # вот что реши проблему
            'collation': 'utf8mb4_unicode_ci'
        }

    def get_all_items(self):
        try:
            conn = mysql.connector.connect(**self.config)
            cursor = conn.cursor(dictionary=True)
            cursor.execute('SELECT name FROM items')
            items = cursor.fetchall()
            cursor.close()
            conn.close()
            return items
        except Exception as e:
            print(f"Error: {e}")
            return []
```
Настройки подключения к MySQL: адрес, порт, учётные данные берутся из переменных окружения (с резервными значениями на случай, если переменные не заданы). Главное – явно указана кодировка utf8mb4, чтобы русский текст не искажался.


---
## Часть I. Docker
Первоначально Docker отсутствовал в системе. Была выполнена установка согласно официальной инструкции для Ubuntu 24.04:
```sh
# Добавление официального репозитория Docker
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Для запуска Docker без прав суперпользователя текущий пользователь был добавлен в группу `docker`:
```sh
sudo usermod -aG docker $USER
newgrp docker
```
---

### 1. Сборка Docker-образа
```sh
docker build -t lab-web .
```
<details> <summary>Вывод команды</summary>
[+] Building 2.7s (11/11) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load metadata for docker.io/library/python:3.9-slim
 => [1/6] FROM docker.io/library/python:3.9-slim
 => [2/6] WORKDIR /app
 => [3/6] RUN apt-get update && apt-get install -y build-essential
 => [4/6] COPY app/requirements.txt .
 => [5/6] RUN pip install --no-cache-dir -r requirements.txt
 => [6/6] COPY app/ .
 => exporting to image
 => => naming to docker.io/library/lab-web:latest
 </details>
 
---

### 2. Запуск контейнера

```sh
docker run -d --name lab_container -p 5000:5000 lab-web
```

Пояснение флагов:

- -d – detached mode (контейнер работает в фоне)
-  --name lab_container – присваивает контейнеру фиксированное имя
- -p 5000:5000 – пробрасывает порт 5000 хоста на порт 5000 внутри контейнера


---

### 3. Копирование файла README.md в контейнер.

Скопировали командой: `docker cp README.md lab_container:/home/`


### 4. Подключение к терминалу контейнера и проверка
Команда `docker exec -it lab_container /bin/bash`
- exec – выполняет команду внутри уже запущенного контейнера.
- -it – интерактивный режим с TTY (позволяет взаимодействовать с оболочкой).
Внутри контейнера:
```sh
ls -la /home/
```
Результат:
```txt
 root@c2008864c83b:/app# ls -la /home/
total 8
drwxr-xr-x 1 root root 4096 Jun  8 14:17 .
drwxr-xr-x 1 root root 4096 Jun  8 14:17 ..
-rw-rw-r-- 1 1000 1000    0 Jun  8 14:17 README.md
```
Файл README.md успешно скопирован в каталог /home/ контейнера.

---

### 5. Выйдите из интерактивного режима.
Команда `exit`

---
### 6. Остановите контейнер с приложением.
Останавливаем командой `docker stop lab_container`. Смотрим, что действительно остановилось командой ` docker ps -a`
```txt
CONTAINER ID   IMAGE     COMMAND           CREATED         STATUS                        PORTS     NAMES
18cbb89d9b18   lab-web   "python app.py"   2 minutes ago   Exited (137) 14 seconds ago             lab_container
```
---
## Часть II. Docker compose
### 1. Файл docker-compose.yml
Файл создан (представлен выше).

---
### 2. Запустите связку web-приложение - БД.
Команды:
```sh
docker compose down -v          # удаление старых контейнеров и томов
docker compose up -d --build    # сборка и запуск в фоне
```
- down -v – останавливает контейнеры и удаляет связанные тома (volumes), гарантируя чистую инициализацию.
- up -d – запускает сервисы в фоновом режиме.
- --build – принудительно пересобирает образ приложения перед запуском.


<details> <summary>Вывод команд </summary>
    
[+] down 2/2
 ✔ Network lab_docker_default Removed
 ✔ Volume lab_docker_db_data  Removed

[+] Building 2.2s (13/13) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load metadata for docker.io/library/python:3.9-slim
 => [1/6] FROM docker.io/library/python:3.9-slim
 => [2/6] WORKDIR /app
 => [3/6] RUN apt-get update && apt-get install -y build-essential
 => [4/6] COPY app/requirements.txt .
 => [5/6] RUN pip install --no-cache-dir -r requirements.txt
 => [6/6] COPY app/ .
 => exporting to image
 => => naming to docker.io/library/lab_docker-app:latest

[+] up 5/5
 ✔ Image lab_docker-app       Built
 ✔ Network lab_docker_default Created
 ✔ Volume lab_docker_db_data  Created
 ✔ Container mysql_db         Healthy
 ✔ Container lab_docker       Started 
    
</details>

### 3-4. Проверка работы приложения через браузер
Веб-приложение полностью работает, все контейнеры запущены, база данных подключена. Это подтверждается выводом curl (Почему то не прикрепляется фото в гитхабе, очень долго мучился, смотретёл в гугле, сказали что бывают такие проблемы со стороны гитхаба, поэтому проверяю через терминал)

Вывод:
```
ubuntu@ubuntu:~/Desktop/artem20308/workspace/projects/lab08$ curl http://localhost:5000
<!DOCTYPE html>
<html>
<head><title>Задачи</title><meta charset="utf-8"></head>
<body>
<h1>Список из Базы Данных</h1>
<ul>
<li>Пример 1</li>
<li>Пример 2</li>
</ul>
</body>
</html>
```
<img width="1606" height="925" alt="Снимок 08 06 2026 в 18 04" src="https://github.com/user-attachments/assets/a0ae2464-a42c-4187-9dc2-59f35325ad30" />

