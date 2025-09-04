# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет
сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и
Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и
Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).
---
## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его
установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции
будут его активно использовать.
---
## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
  docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
  docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
  ocker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по
адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).
---
## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не
требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
    git pull
    docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции
схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие
миграции, то команда их применит:

```shell
  docker compose run --rm web ./manage.py migrate

  Running migrations:
    No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой
библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
  docker compose build web
```
Аналогичным образом можно удалять библиотеки из зависимостей.

---
# Переменные окружения

Образ с Django считывает настройки из переменных окружения.  
Создайте файл `.env` в корневом каталоге и запишите туда данные в формате: `ПЕРЕМЕННАЯ=значение`

- **`SECRET_KEY`** — обязательная секретная настройка Django.  
   Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно.  
  [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key)
- **`DEBUG`** — настройка Django для включения отладочного режима.  
  Принимает значения `TRUE` или `FALSE`.  
  [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG)

- **`ALLOWED_HOSTS`** — настройка Django со списком разрешённых адресов.  
  Если запрос прилетит на другой адрес, то сайт ответит ошибкой **400 Bad Request**.  
  Можно перечислить несколько адресов через запятую, например: `127.0.0.1,192.168.0.1,site.test`
  [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts)
- **`DATABASE_URL`** — адрес для подключения к базе данных PostgreSQL.  
  Другие СУБД не поддерживаются.  
  [Формат записи](https://github.com/jacobian/dj-database-url#url-schema)
---
# Kubernetes

### Локальный запуск с использованием кластера Minikube

   Для запуска необходимы:
   
- [См. документацию Docker Desktop](https://www.docker.com/get-started/)
   
- [См. документацию сubectl](https://kubernetes.io/docs/tasks/tools/)
   
- [См. документацию minikube](https://kubernetes.io/docs/tasks/tools/)


1. **Запустить кластер Minikube:**

    ```sh
      minikube start --driver=virtualbox
    ```
    * *Вместо docker можно использовать другой драйвер (например, docker), если он у вас настроен:*

        ```sh
        minikube start --driver=vdocker
        ```
2. **Передача чувствительных данных:**

   Создайте файл `secrets.yaml`, заменив значения переменных на свои (*см. раздел переменные окружения*):

    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: django-secrets
    type: Opaque
    stringData:
      DEBUG: "False"
      SECRET_KEY: "your_secret_key"
      DATABASE_URL: "your_database_url"
      ALLOWED_HOSTS: "allowed_hosts"
    ```

   Примените:
    ```sh
      kubectl apply -f secrets.yaml
    ```

3. **Запуск приложения:**
    - Запуск базы данных PostgreSQL: (в разработке)
    - Запуск Django-приложения и сервиса:
        ```sh
          kubectl apply -f django-deployment.yaml
          kubectl apply -f django-service.yaml
        ```
4. **Выполнить миграции и создать суперпользователя:**
    ```sh
      kubectl exec -it deploy/django-deployment -- python manage.py migrate # создаём/обновляем таблицы в БД
    
      kubectl exec -it deploy/django-deployment -- python manage.py createsuperuser # создаём суперпользователя
    ```
5. **Проверить доступность сайта:**
    ```sh
      minikube service django-service --url
    ```

6. **(Опционально) Настроить Ingress**
    1. Отредактируйте файл `kubernetes/ingress-hosts.yaml`, укажите свой домен:
        ```
        ...
        spec:
          rules:
            - host: your-domain.test
        ...
        ```
    2. При локальной разработке:
        - Узнайте внешний IP Minikube:
            ```sh
            minikube ip
            ```
          пример ответа:
            ```
            192.168.59.100
            ```
        - Добавьте в файл `host` вашей ОС строку:

            ```
            192.168.59.100   your-domain.test
            ```
    3. Примените манифест Ingress::
        ```sh
          kubectl apply -f kubernetes/ingress-hosts.yaml
        ```