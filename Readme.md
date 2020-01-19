### Установка и настройка Gitlab внутри Docker
[Оригинал статьи](https://rascal.su/blog/2016/09/18/%D1%80%D0%B0%D0%B7%D0%B2%D0%BE%D1%80%D0%B0%D1%87%D0%B8%D0%B2%D0%B0%D0%B5%D0%BC-gitlab-%D0%BD%D0%B0-%D0%B1%D0%B0%D0%B7%D0%B5-docker/)


# 1. Создаем файл docker-compose.yml
    Перманентная информация хранится на хосте в каталоге /export/containers.

# №2. Создаем контейнеры описанной конфигурации:

```
$ docker-compose create
Creating redis
Creating postgresql
Creating gitlab_app
Creating nginx
```


## PostgreSQL

Для начала настроим PostgreSQL. Для этого запустим контейнер:

```
$ docker start postgresql
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
aeab9cf821a3        postgres:latest     "/docker-entrypoint.s"   57 seconds ago      Up 5 seconds        5432/tcp            postgresql
```

Создадим необходимых для работы пользователя и БД

```
$ docker exec -i -t postgresql /bin/bash
$ su - postgres
$ createuser -P git
$ createdb -O git gitlabhq_production
$ echo 'CREATE EXTENSION pg_trgm;' | psql gitlabhq_production
```

После конфигурации контейнер можно остановить:

```
$ docker stop postgresql
```



## Nginx

Нам потребуются конфиги nginx, можно использовать готовые:

```
$ curl https://raw.githubusercontent.com/R4scal/docker-gitlab/master/nginx-conf/nginx.conf > /export/containers/nginx-conf/nginx.conf
$ curl https://raw.githubusercontent.com/R4scal/docker-gitlab/master/nginx-conf/gitlab-http.conf > /export/containers/nginx-conf/gitlab-http.conf
```



## Gitlab

Официальный сайт говорит, что при запуске контейнера конфигурацию можно передать через переменные окружения. Для этого в секцию сервиса gitlab_app необходимо добавить строки вида:

environment:
  GITLAB_OMNIBUS_CONFIG: |
    external_url 'http://git'

Но для меня такой способ не сработал. При первом запуске происходит создание файла конфигурации gitlab.rb, который необходимо отредактировать вручную указав параметры:

```
external_url 'http://git'
gitlab_rails['db_adapter'] = "postgresql"
gitlab_rails['db_database'] = "gitlabhq_production"
gitlab_rails['db_username'] = "git"
gitlab_rails['db_password'] = "T0pS3cr3T"
gitlab_rails['db_host'] = "postgresql"
gitlab_rails['db_port'] = 5432
gitlab_rails['redis_host'] = "redis"
gitlab_rails['redis_port'] = 6379
gitlab_rails['redis_database'] = 0
gitlab_workhorse['enable'] = true
gitlab_workhorse['listen_network'] = "tcp"
gitlab_workhorse['listen_addr'] = "0.0.0.0:8081"
gitlab_workhorse['auth_backend'] = "http://localhost:8080"
unicorn['listen'] = '127.0.0.1'
unicorn['port'] = 8080
postgresql['enable'] = false
redis['enable'] = false
nginx['enable'] = false
```


## Запускаем все контейнеры:

```
$ docker-compose up
```

После этого все должно работать. Необходимо только открытия в браузере http://<адрес_сервера>:8080 и на экране можно увидеть предложение задать пароль. После установки пароля и авторизации попадаешь в веб-интерфейс и панель администратора.
