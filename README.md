# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Запуск приложения в Kubernetes

### 1. Подготовка к развертыванию в Kubernetes
Необходимо установить:
1. Для локальной разработки устанавливаем [Minikube ](https://minikube.sigs.k8s.io/docs/)
2. Helm [инструкция по установке](https://helm.sh/)
3. Ingress контроллер, например по [ссылке](https://github.com/projectcontour/contour)

### 1. Билдим образ приложения в minikube

Выполняем команду:
```commandline
minikube image build backend_main_django/ . -t django_app
```

### 2. Устанавливаем переменные окружения

Для создания переменных окружения  с настройками в Kubernetes, используем `configMap` 
(для хранения секретных данных будем использовать Secrets, инструкция ниже).
```commandline
kubectl create configmap env-config 
    --from-literal=DEBUG=True
    --from-literal=ALLOWED_HOSTS=''
```

Для изменения данного `configMap` используем команду:
```commandline
kubectl edit configmap env-config
```

Для хранения секретных данных таких как DATABASE_URL, воспользуемся Secrets. Значения секретных данных берем из файла `.env` 

```commandline
kubectl create secret generic env-config 
    --from-literal=DATABASE_URL=''
```

Создаем Secret c ssl сертификатом:
```commandline
kubectl create secret generic postgresql-ssl --from-file=root.crt
```

### 3. Настройка и запуск БД
Сначала устанавливаем БД Postgresql c помощью команды
```commandline
helm install djangoappdb oci://registry-1.docker.io/bitnamicharts/postgresql
 --set global.postgresql.postgresqlUsername=[имя пользователя как в .env]
 --set global.postgresql.postgresqlPassword=[пароль как в .env] 
 --set global.postgresql.postgresqlDatabase=[Название POD`а]
```

Далее смотрим название сервиса с postgresql командой `kubectl get svc`, вносим название сервиса Postgresql в configMap
в переменную `DATABASE_URL`

### 4. Запускаем приложение командой: 

Файл `Chart-djangoapp/values.yaml` содержит переменные для запуска приложения в Helm. Сейчас только название
image.

```commandline
helm install djangoapp Chart-djangoapp/
```

При первом деплое необходимо создать суперюзера командой:
```commandline
kubectl exec -it djnangoapp -- python manage.py createsuperuser
```

### 5. Для доступа к сайту по домену используем IP адрес сервиса
Данный ip c приложением который можно узнать командой `kubectl get svc` 
и вносим этот IP в hosts


### 6. Как задеплоить код в кластер 

В директории `deploy/yc-sirius/edu-desperate-sinuossi` лежат файлы настроек для PODа и Service

