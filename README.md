# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию в docker-compose

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web python3 ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web python3 ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Как запустить dev-версию в minikube

Создать docker образ

```shell-session
$ minikube image build -t django_app . 
```

Создать конфиг (например в папке config внутри папки kubernetes)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config-v1 # в названии номер версии для дальнейшего обновления
  labels:
    name: django
    instance: django-k8s
    version: "1.0.0"
    component: config
    part-of: server
    managed-by: k8s
data:
  SECRET_KEY: "" # смотреть секцию при переменные окружения 
  DEBUG: "TRUE" # смотреть секцию при переменные окружения
  DATABASE_URL: "" # смотреть секцию при переменные окружения
  ALLOWED_HOSTS: "*"
```

Загрузите конфиг, деплоймент и сервис в minikube

```shell-session
$ kubectl apply -f .\kubernetes\config\configmap.yaml
$ kubectl apply -f .\kubernetes\deployment.yaml
$ kubectl apply -f .\kubernetes\service.yaml
```

## Как обновить dev-версию в minikube

Если меняются переменные окружения, то измените версию в названии конфига (metadata name), а затем
загрузите новую версию конфига в minikube

```shell-session
$ kubectl apply -f .\kubernetes\config\configmap.yaml
```

Если изменился образ и/или изменился конфиг, то измените название конфига при его изменении, а затем
загрузите снова deployment, при этом обновление будет без остановки сервиса

```shell-session
$ kubectl apply -f .\kubernetes\deployment.yaml
```

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).