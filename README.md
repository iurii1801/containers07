# Лабораторная работа №7: Создание многоконтейнерного приложения

## Цель работы

Ознакомиться с работой многоконтейнерного приложения на базе `docker-compose`.

## Задание

Создать PHP-приложение на базе трёх контейнеров: `nginx`, `php-fpm`, `mariadb`, используя `docker-compose`.

## Ход работы

### 1. Клонирован пустой Git-репозиторий `containers07`

```bash
git clone git@github.com:iurii1801/containers07.git
```

![image](https://i.imgur.com/Jcw6hEN.png)

### 2. Созданы папки `nginx/` и `mounts/site/`, в которой будет сайт на **PHP**, созданный в рамках предмета по **PHP**.

```powershell
# Создание папки для конфигурации nginx
New-Item -ItemType Directory -Path "nginx"

# Создание вложенной папки для сайта
New-Item -ItemType Directory -Path "mounts/site" -Force
```

![image](https://i.imgur.com/fADo0z1.png)

### 3. Создание файлов `.gitignore` и `README.md`

```powershell
# Создание файла .gitignore
New-Item -Name ".gitignore" -ItemType "File"
```

![image](https://i.imgur.com/vsW6pjW.png)

```powershell
# Создание файла README.md
New-Item -Name "README.md" -ItemType "File"
```

![image](https://i.imgur.com/M5nHCFb.png)

### 4. В `.gitignore` добавлено

```sh
# Ignore files and directories
mounts/site/*
```

![image](https://i.imgur.com/fge7ows.png)

### 5. В директории `containers07` добавлен файл `nginx/default.conf`

```nginx
New-Item -Path "nginx" -Name "default.conf" -ItemType "File"
```

![image](https://i.imgur.com/GJ5KJLj.png)

со следующим содержимым

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.php;
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    location ~ \.php$ {
        fastcgi_pass backend:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

![image](https://i.imgur.com/8hwrDQg.png)

### 6. Создание переменных окружения

#### 6.1 Создание файла `mysql.env`

Создан файл `mysql.env` в корне проекта. Он используется для хранения конфигурационных параметров базы данных MySQL. Такие параметры, как имя базы данных, имя пользователя, пароль и root-пароль, передаются как переменные окружения при запуске контейнера database. Это позволяет удобно и безопасно задавать настройки без жёсткого кодирования значений в docker-compose.yml. Также такой подход облегчает переносимость и масштабируемость проекта при использовании разных окружений (например, dev, staging, production):

```env
MYSQL_ROOT_PASSWORD=secret
MYSQL_DATABASE=app
MYSQL_USER=user
MYSQL_PASSWORD=secret
```

![image](https://i.imgur.com/noa6QPc.png)

#### 6.2 Создание файла `app.env`

Создан файл `app.env` в корне проекта. Он предназначен для хранения пользовательских переменных окружения, которые можно переиспользовать в нескольких сервисах. Это удобно для настройки общих параметров, таких как версии приложения, режим работы, параметры путей и другие конфигурационные данные, которые должны быть доступны одновременно как `frontend`, так и `backend`. Такой подход способствует более чистой архитектуре, лучшей управляемости и облегчает развертывание приложения в разных средах (например, dev, test, prod).

```env
APP_VERSION=1.0.0
```

![image](https://i.imgur.com/El5tHEz.png)

Этот файл содержит пользовательскую переменную окружения `APP_VERSION`, которую можно использовать внутри контейнеров для отображения информации о версии приложения, ведения логов или другой конфигурации. Переменная подключается в `docker-compose.yml` к сервисам `frontend` и `backend` через параметр `env_file`. Это позволяет централизованно управлять значениями и использовать их в конфигурациях или PHP-коде (например, через `getenv('APP_VERSION')`).

### 7. Создание docker-compose.yml в директории `containers07`

Файл `docker-compose.yml` в корне проекта:

```yaml
version: '3.9'

services:
  frontend:
    image: nginx:1.19
    volumes:
      - ./mounts/site:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
    networks:
      - internal
    env_file:
      - app.env

  backend:
    image: php:7.4-fpm
    volumes:
      - ./mounts/site:/var/www/html
    networks:
      - internal
    env_file:
      - mysql.env
      - app.env

  database:
    image: mysql:8.0
    env_file:
      - mysql.env
    networks:
      - internal
    volumes:
      - db_data:/var/lib/mysql

networks:
  internal: {}

volumes:
  db_data: {}
```

Файл `docker-compose.yml` в корне проекта отвечает за описание и координацию всех контейнеров, участвующих в многоконтейнерном приложении. В данном случае он описывает три сервиса: `frontend` (`nginx`), `backend` (`php-fpm`) и `database` (`MySQL`). Каждому сервису назначаются тома, переменные окружения, порты, сетевые настройки и подключение конфигурационных файлов. Использование `docker-compose` позволяет запускать и управлять всем окружением одной командой, упрощая процесс разработки, тестирования и развертывания.

![image](https://i.imgur.com/87UCjvq.png)

### 8. Запуск контейнеров и тестирование

```bash
docker-compose up -d
```

Команда инициирует процесс сборки и запуска всех сервисов, указанных в конфигурации. Ключ `-d` (или `--detach`) означает, что контейнеры будут работать в фоновом режиме, не блокируя терминал. После выполнения этой команды создаётся изолированная среда, где каждый контейнер запускается с указанными параметрами: монтируются тома, применяются переменные окружения, активируются сетевые настройки и пробрасываются порты. Это позволяет в один шаг развернуть полностью функционирующее многоконтейнерное приложение.

![image](https://i.imgur.com/AyjZWWa.png)

Контейнеры успешно запущены и взаимодействуют между собой. Сайт работает по адресу: [http://localhost](http://localhost).

---

## Вопросы и ответы

### В каком порядке запускаются контейнеры?

Контейнеры запускаются независимо друг от друга, так как `docker-compose` по умолчанию не управляет порядком готовности сервисов. Однако их порядок можно частично контролировать с помощью параметра `depends_on`. Например, можно указать, что контейнер `backend` зависит от `database`, но это не гарантирует, что база будет готова к подключению. Поэтому необходимо реализовать задержку или проверку соединения с БД в коде или скриптах инициализации.

### Где хранятся данные базы данных?

Данные MySQL сохраняются внутри тома `db_data`, который подключается к контейнеру базы данных по пути `/var/lib/mysql`. Это обеспечивает сохранность данных при перезапуске контейнеров или при пересборке проекта. Docker volume работает независимо от жизненного цикла контейнера и остаётся доступным, даже если контейнер был удалён.

### Как называются контейнеры проекта?

Docker Compose формирует имена контейнеров из названия директории проекта и названия сервиса. Например, при запуске в папке `containers07` имена контейнеров будут следующими: `containers07_frontend_1`, `containers07_backend_1`, `containers07_database_1`. Это позволяет точно понимать структуру и принадлежность каждого контейнера.

### Вам необходимо добавить ещё один файл `app.env` с переменной окружения `APP_VERSION` для сервисов `backend` и `frontend`. Как это сделать?

Для этого нужно выполнить следующие шаги:

- Создание файла `app.env` в корне проекта

```powershell
New-Item -Name "app.env" -ItemType "File"
```

- Добавление переменной `APP_VERSION` в файл

```powershell
Set-Content -Path "app.env" -Value "APP_VERSION=1.0.0"
```

- Подключение `app.env` к `frontend` и `backend` в `docker-compose.yml`

Открой `docker-compose.yml` в любом редакторе и добавь (если ещё не добавлено) под соответствующими сервисами:

**Пример для `frontend`**

```yaml
    env_file:
      - app.env
```

**Пример для `backend`**

```yaml
    env_file:
      - mysql.env
      - app.env
```

---

## Выводы

В результате лабораторной работы было развёрнуто многоконтейнерное приложение с использованием Docker Compose. Все три контейнера — nginx, php-fpm и mariadb — были успешно настроены и связаны через внутреннюю сеть. Конфигурация nginx была переопределена для правильной обработки PHP-файлов, а в контейнерах были использованы переменные окружения, включая пользовательскую переменную `APP_VERSION`, подключённую через отдельный файл `app.env`.

Проект протестирован локально, сайт корректно отображается при обращении к `http://localhost`, PHP-код обрабатывается, и взаимодействие с базой данных проходит успешно. Работа показала важность использования конфигурационных файлов, переменных окружения и volumes для управления состоянием приложения. Полученные навыки углубляют понимание контейнеризации и подготовки современных web-приложений к развёртыванию.

---

## Библиография

1. [Официальная документация Docker](https://docs.docker.com/)
2. [Документация Docker Compose](https://docs.docker.com/compose/)
3. [DockerHub: php:7.4-fpm](https://hub.docker.com/_/php)
4. [DockerHub: nginx](https://hub.docker.com/_/nginx)
5. [DockerHub: mysql](https://hub.docker.com/_/mysql)
6. [PHP: getenv – Официальная документация](https://www.php.net/manual/ru/function.getenv.php)
7. [Nginx: FastCGI конфигурация](https://nginx.org/ru/docs/http/ngx_http_fastcgi_module.html)