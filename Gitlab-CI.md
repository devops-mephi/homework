# Gitlab-CI

## Continius integration

Continius integration (CI) — практика, при которой код помещается в общий репозиторий, а на каждое изменение в этом репозитории автоматически прогоняется сборка/тестирование написанного кода.

Gitlab CI — встроенный в Gitlab инструмент для реализации практики CI.
Он состоит из двух частей: Runner'ы и файл .gitlab-ci.yml

Runner — машина, желательно отдельная (мы будем ставить в ту же виртуальную машину, так как у нас нет столько ресурсов на ноутбуках), которая занимается выполнением задач, описанных в .gitlab-ci.yml файле.
Установим runner'а внутри виртуальной машины vagrant

1. Подключение репозитория
```
[vagrant@localhost banners]$ curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
```

2. Установка

```
[vagrant@localhost banners]$ sudo yum install -y gitlab-runner
```

3. Регистрация

Откройте настройки проекта Settings - CI / CD

![Runners](images/runners.png)

Запустите
```
sudo gitlab-runner register
```

Он будет задавать вопросы, отвечайте тем, что видите на экране настроек проекта
```
Runtime platform                                    arch=amd64 os=linux pid=29782 revision=4745a6f3 version=11.8.0
Running in system-mode.                            
                                                   
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://127.0.0.1/
Please enter the gitlab-ci token for this runner:
7pNpywCPRfNdNgwMBij9
Please enter the gitlab-ci description for this runner:
[localhost.localdomain]: vagrant
Please enter the gitlab-ci tags for this runner (comma separated):
vagrant
Registering runner... succeeded                     runner=7pNpywCP
Please enter the executor: shell, ssh, virtualbox, docker+machine, kubernetes, docker, docker-ssh, parallels, docker-ssh+machine:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
[vagrant@localhost banners]$
```

На вопрос, какой executor выбрать, ответим "shell". Это значит, что команды от gitlab-ci будут выполнены прямо в shell'е на машине, на которой установлен runner.
Есть другие опции вроде виртуальных машин (будет поднята машина на каждую сборку) или docker (будет создан контейнер под каждую сборку)

После окончания регистрации на странице настроек проекта должен быть виден настроенный runner.

![Runner activated](images/runner_activated.png)

4. Добавление прав пользователю gitlab-runner.
Мы будем хотеть запускать команды докера из раннера. Поэтому нужно добавить его в группу "docker"

```
sudo usermod -aG docker gitlab-runner
```

5. Теперь добавим файл .gitlab-ci.yml в корень проекта
```
stages:
  - build

build:
  stage: build
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - vagrant
```

6. Когда мы запушим изменение из пункта 5, должна автоматически запуститься сборка 

![Gitlab CI job](images/gitlab-ci-job.png)


## Docker runner

Схема с shell-раннером хороша своей простотой, но у неё есть несколько недостатков: нельзя делать параллельные запуски (так как они будут друг друга аффектить) и существует зависимость от состояния машины. Например, если прошлая сборка что-то положила в кеш докера, то новая будет брать из этого кеша. Можно запускать на каждую сборку виртуальную машину, но это довольно тяжело. Будем запускать каждую сборку в отдельном докер-контейнере.
Но цель нашей сборки — собрать докер контейнер, то есть нам потребуется запускать докер внутри докера. К счастью, существует возможность так сделать.

1. Настроить такой runner уже придется вручную:
```
sudo gitlab-runner register -n \
--url http://gitlab.mephi/ \
--registration-token J_HKEz_YhzwApzCjtb5T \
--description "Vagrant Docker" \
--tag-list docker_in_docker \
--executor docker \
--docker-image "docker:stable" \
--docker-privileged
```

Итого у нас получилось 2 раннера на одной машине:

![Gitlab CI two runners](images/gitlab-two-runners.png)

2. Допишем .gitlab-ci файл:

```
stages:
  - build

build:
  stage: build
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - vagrant
    
build_in_docker:
  services:
    - docker:dind
  stage: build
  variables:
    DOCKER_HOST: tcp://docker:2375/
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - docker_in_docker
```

3. Запушим, попробуем запустить:

```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on Vagrant Docker jGJ96CGz
Using Docker executor with image docker:stable ...
Starting service docker:dind ...
Pulling docker image docker:dind ...
Using docker image sha256:85e924caedbd3e5245ad95cc7471168e923391b22dcb559decebe4a378a06939 for docker:dind ...
Waiting for services to be up and running...
Pulling docker image docker:stable ...
Using docker image sha256:639de9917ae1f2e4a586893c9a6ea3f21fd774bc4037184ecac35f3153a293b5 for docker:stable ...
Running on runner-jGJ96CGz-project-2-concurrent-0 via st91.i...
Cloning repository...
Cloning into '/builds/devops-mephi/banners'...
fatal: unable to access 'http://gitlab-ci-token:xxxxxxxxxxxxxxxxxxxx@gitlab.mephi/devops-mephi/banners.git/': Could not resolve host: gitlab.mephi
/bin/bash: line 65: cd: /builds/devops-mephi/banners: No such file or directory
ERROR: Job failed: exit code 1
```

Хм, получилось так, что докер внутри докера пытается обратиться по gilab.mephi и не знает, куда пойти, потому что нет записи в /etc/hosts.

4. Добавить эту запись можно следующим образом в конфиг gitlab-runner'а

```
sudo vi /etc/gitlab-runner/config.toml
```

допишем следующую строчку в runners.runners.docker
```
    extra_hosts = ["gitlab.mephi:127.0.0.1"]
```
и перезапустим раннеры

```
[vagrant@localhost banners]$ sudo gitlab-runner restart
Runtime platform                                    arch=amd64 os=linux pid=18901 revision=4745a6f3 version=11.8.0
```

5. Пробуем заново:

теперь новая ошибка
```
Cloning into '/builds/devops-mephi/banners'...
fatal: unable to access 'http://gitlab-ci-token:xxxxxxxxxxxxxxxxxxxx@gitlab.mephi/devops-mephi/banners.git/': Failed to connect to gitlab.mephi port 80: Connection refused
/bin/bash: line 65: cd: /builds/devops-mephi/banners: No such file or directory
ERROR: Job failed: exit code 1
```

Подсоединиться к хосту удалось, но там нет 80 порта. Хм.
Разгадка такова: для докера внутри докера 127.0.0.1 это не локальная машина vagrant, а контейнер. Нам нужно прописать IP именно машины с vagrant внутри docker-сети.
Узнать нужный IP можно следующим образом:

```
[vagrant@st91 banners]$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:75:dc:3d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 79731sec preferred_lft 79731sec
    inet6 fe80::5054:ff:fe75:dc3d/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:c6:c4:39:a1 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c6ff:fec4:39a1/64 scope link
       valid_lft forever preferred_lft forever
```

Нас интересует сеть docker0. Пропишем этот IP: 172.17.0.1

```
sudo vi /etc/gitlab-runner/config.toml
```

в runners.runners.docker
```
    extra_hosts = ["gitlab.mephi:172.17.0.1"]
```

6. Пробуем еще раз. Успех!

![Gitlab CI success](images/gitlab-docker-success.png)

PS: если бы наш гитлаб жил на нормальной отдельной машине с собственным настоящим доменным именем, это шаманство с IP адресами бы не потребовалось.

7. Отключим пока автоматический запуск этой сборки (потому что она слишком долгая и будет нам мешаться)

Допишем в .gitlab-ci.yaml строчку "when: manual"

```
stages:
  - build

build:
  stage: build
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - vagrant
    
build_in_docker:
  services:
    - docker:dind
  stage: build
  variables:
    DOCKER_HOST: tcp://docker:2375/
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - docker_in_docker
  when: manual
```

Видим, что теперь сборка автоматически не запустилась, а появилась кнопка, по которой её можно запустить

![Gitlab CI manual](images/gitlab-manual.png)

## Unit-тестирование с gitlab

Теперь мы будем делать сборку, которая возьмет собранный на предыдущем шаге образ (banners_web:$CI_BUILD_REF_NAME) и прогонит на нем unit-тесты для Django.

1. Начнем с того, что просто создадим сборку, которая будет запускаться в контейнере, запущенном из образа banners_web

.gitlab-ci.yaml
```
stages:
  - build
  - test

build:
  stage: build
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - vagrant
    
build_in_docker:
  services:
    - docker:dind
  stage: build
  variables:
    DOCKER_HOST: tcp://docker:2375/
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - docker_in_docker
  when: manual

unit-test:
  image: banners_web:$CI_BUILD_REF_NAME
  stage: test
  tags:
    - docker_in_docker
  script:
    - whoami
    - pwd
    - python manage.py test
```

2. Получили следующую ошибку:

```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on Vagrant Docker pvb9SWyA
Using Docker executor with image banners_web:ban-001-ci ...
Pulling docker image banners_web:ban-001-ci ...
ERROR: Job failed: Error response from daemon: pull access denied for banners_web, repository does not exist or may require 'docker login' (executor_docker.go:168:1s)
```
Что же произошло: по умолчанию для docker-runner'ов установлена следующая политика pull_policy=always. Она означает, что докер-образ ВСЕГДА pull'ится из реджистри. Для нашего случая (мы собрали образ локально и хотим его заиспользовать) она не годится, поменяем на if-not-present (лезть в интернет, только если нет локально)

Управляется это в конфигурации runner'ов:
```
[vagrant@localhost banners]$ sudo vi /etc/gitlab-runner/config.toml
```

допишем в блок runnsers.docker
```
pull_policy = "if-not-present"
```

перезапустим
```
[vagrant@localhost banners]$ sudo gitlab-runner restart
Runtime platform                                    arch=amd64 os=linux pid=26292 revision=4745a6f3 version=11.8.0
```

3. Попробуем запустить сборку еще раз:

```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on Vagrant Docker pvb9SWyA
Using Docker executor with image banners_web:ban-001-ci ...
Using locally found image version due to if-not-present pull policy
Using docker image sha256:4cc34c36f44746f7797ed5bc88bb83028cbb458e74244dd11d567eb31356e047 for banners_web:ban-001-ci ...
Running on runner-pvb9SWyA-project-1-concurrent-0 via localhost.localdomain...
Fetching changes...
HEAD is now at 60cec49 Update .gitlab-ci.yml
From http://gitlab.mephi/devops-mephi/banners
   60cec49..4f17bcc  ban-001-ci -> origin/ban-001-ci
Checking out 4f17bcc9 as ban-001-ci...
Skipping Git submodules setup
$ whoami
root
$ pwd
/builds/devops-mephi/banners
$ python manage.py test
Creating test database for alias 'default'...
Traceback (most recent call last):
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/base/base.py", line 216, in ensure_connection
    self.connect()
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/base/base.py", line 194, in connect
    self.connection = self.get_new_connection(conn_params)
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/mysql/base.py", line 227, in get_new_connection
    return Database.connect(**conn_params)
  File "/usr/local/lib/python3.7/site-packages/MySQLdb/__init__.py", line 84, in Connect
    return Connection(*args, **kwargs)
  File "/usr/local/lib/python3.7/site-packages/MySQLdb/connections.py", line 164, in __init__
    super(Connection, self).__init__(*args, **kwargs2)
MySQLdb._exceptions.OperationalError: (2005, "Unknown MySQL server host 'db' (-2)")
```

Ага, получить образ получилось!
Что произошло дальше: gitlab ci создал рабочую папку /builds/devops-mephi/ и в неё склонировал репозиторий.
Нам это поведение в данном случае совсем не нужно, ведь у нас уже есть готовый образ со всем кодом и прочим.

4. Отключить это можно дописыванием
```
  variables:
    GIT_STRATEGY: none
```
в описание сборки gitlab-ci.yml

Получается вот так:
```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on Vagrant Docker pvb9SWyA
Using Docker executor with image banners_web:ban-001-ci ...
Using locally found image version due to if-not-present pull policy
Using docker image sha256:4d296db4f0c32e5549aaca3cde310e5ca0fce3f4762928b62ea5dca112e3fa01 for banners_web:ban-001-ci ...
Running on runner-pvb9SWyA-project-1-concurrent-0 via localhost.localdomain...
Skipping Git repository setup
Skipping Git checkout
Skipping Git submodules setup
$ whoami
root
$ pwd
/builds/devops-mephi/banners
$ python manage.py test
python: can't open file 'manage.py': [Errno 2] No such file or directory
ERROR: Job failed: exit code 1
```

Папка осталась, но нам нужно запускать python manage.py из папки /code/ где лежит код.

5. Поменяем скрипт на такой (здесь и далее я не буду больше приводить весь .gitlab-ci.yml файл: 
```
script:
    - whoami
    - cd /code/
    - python manage.py test
```

Получаем:
```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on Vagrant Docker pvb9SWyA
Using Docker executor with image banners_web:ban-001-ci ...
Using locally found image version due to if-not-present pull policy
Using docker image sha256:48cc1be4487446782c7844e8a83e70caa0709d281108f1d66a2ce36e2a3eaa8a for banners_web:ban-001-ci ...
Running on runner-pvb9SWyA-project-1-concurrent-0 via localhost.localdomain...
Skipping Git repository setup
Skipping Git checkout
Skipping Git submodules setup
$ whoami
root
$ cd /code/
$ python manage.py test
Creating test database for alias 'default'...
Traceback (most recent call last):
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/base/base.py", line 216, in ensure_connection
    self.connect()
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/base/base.py", line 194, in connect
    self.connection = self.get_new_connection(conn_params)
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/mysql/base.py", line 227, in get_new_connection
    return Database.connect(**conn_params)
  File "/usr/local/lib/python3.7/site-packages/MySQLdb/__init__.py", line 84, in Connect
    return Connection(*args, **kwargs)
  File "/usr/local/lib/python3.7/site-packages/MySQLdb/connections.py", line 164, in __init__
    super(Connection, self).__init__(*args, **kwargs2)
MySQLdb._exceptions.OperationalError: (2005, "Unknown MySQL server host 'db' (-2)")
```

Отлично, всё запускается, но у нас нет mysql. В mysql django будет создавать тестовую базу данных и гонять на ней тесты.

6. Добавим mysql. Для этого у gitlab-ci есть services. Это описанные сервисы-контейнеры похожие на сервисы в docker-compose.
Опишем mysql для нас:

.gitlab-ci.yml
```
unit-test:
  image: banners_web:$CI_BUILD_REF_NAME
  stage: test
  services:
    - name: mysql:5.7
      alias: db
  tags:
    - docker_in_docker
  variables:
    GIT_STRATEGY: none
    MYSQL_ROOT_PASSWORD: root_1234
  script:
    - whoami
    - cd /code/
    - python manage.py test
```

Важно: мы используем root-пользователя, так как django хочет создавать базы данных под unit-тесты.

запускается сборка и...
```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on Vagrant Docker pvb9SWyA
Using Docker executor with image banners_web:ban-001-ci ...
Starting service mysql:5.7 ...
Using locally found image version due to if-not-present pull policy
Using docker image sha256:ee7cbd482336a1f49b7038527a547f0f25b23a78d46350845e70a1cf7e5a22af for mysql:5.7 ...
Waiting for services to be up and running...
Using locally found image version due to if-not-present pull policy
Using docker image sha256:549a61c5e415b2adc3482cfe69ad551306a8d82cc5f30c11446599a1ec3ac385 for banners_web:ban-001-ci ...
Running on runner-pvb9SWyA-project-1-concurrent-0 via localhost.localdomain...
Skipping Git repository setup
Skipping Git checkout
Skipping Git submodules setup
$ whoami
root
$ cd /code/
$ python manage.py test
Creating test database for alias 'default'...
Traceback (most recent call last):
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/base/base.py", line 216, in ensure_connection
    self.connect()
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/base/base.py", line 194, in connect
    self.connection = self.get_new_connection(conn_params)
  File "/usr/local/lib/python3.7/site-packages/django/db/backends/mysql/base.py", line 227, in get_new_connection
    return Database.connect(**conn_params)
  File "/usr/local/lib/python3.7/site-packages/MySQLdb/__init__.py", line 84, in Connect
    return Connection(*args, **kwargs)
  File "/usr/local/lib/python3.7/site-packages/MySQLdb/connections.py", line 164, in __init__
    super(Connection, self).__init__(*args, **kwargs2)
MySQLdb._exceptions.OperationalError: (1045, "Access denied for user 'developer'@'172.17.0.3' (using password: YES)")
```

Важно отметить:
- gitlab не стал снова скачавать из интернета образ mysql:5.7 потому что мы его уже использовали.
- "Waiting for services to be up and running..." — gitlab более умен, чем docker-compose и ждет запуска сервиса после того, как он стартанул контейнер. Ждет он следующим образом: для каждого порта, описанного в EXPOSE, дожидается, пока он не станет доступен.
- "MySQLdb._exceptions.OperationalError: (1045, "Access denied for user 'developer'@'172.17.0.3' (using password: YES)")" — база данных есть, но нам не удалось к ней подключиться, потому что в настройках прописан пользователь developer.

7. Переопределение настроек.

Что нам нужно сделать: при запуске проекта в сборке unit-test переопределить настройки DATABASES.
Реализуем это с помощью ENVIRONMENT-переменных.

Поправим banners/settings.py следующим образом:
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.environ.get('BANNERS_DATABASE_NAME', 'mephi'),
        'USER':  os.environ.get('BANNERS_DATABASE_USER', 'developer'),
        'PASSWORD': os.environ.get('BANNERS_DATABASE_PASSWORD', '1234'),
        'HOST': os.environ.get('BANNERS_DATABASE_HOST', 'db'),
        'PORT': os.environ.get('BANNERS_DATABASE_PORT', '3306'),
    }
}
```

А в .gitlab-ci.yml пропишем эти переменные
```
variables:
    GIT_STRATEGY: none
    MYSQL_ROOT_PASSWORD: root_1234
    BANNERS_DATABASE_USER: root
    BANNERS_DATABASE_PASSWORD: root_1234
```

И, успех!
```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on Vagrant Docker pvb9SWyA
Using Docker executor with image banners_web:master ...
Starting service mysql:5.7 ...
Using locally found image version due to if-not-present pull policy
Using docker image sha256:ee7cbd482336a1f49b7038527a547f0f25b23a78d46350845e70a1cf7e5a22af for mysql:5.7 ...
Waiting for services to be up and running...
Using locally found image version due to if-not-present pull policy
Using docker image sha256:962408e44881bc705de2184a88316bf6bd344c29efaaa66986a72279a419c226 for banners_web:master ...
Running on runner-pvb9SWyA-project-1-concurrent-0 via localhost.localdomain...
Skipping Git repository setup
Skipping Git checkout
Skipping Git submodules setup
$ whoami
root
$ cd /code/
$ python manage.py test
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
Destroying test database for alias 'default'...

----------------------------------------------------------------------
Ran 0 tests in 0.000s

OK
Job succeeded
```

8. Попробуем написать какие-нибудь тесты

Создадим файл banners/tests.py
и в него напишем следующее:

```
from django.test import TestCase

class ExampleTestCase(TestCase):
    def test_one_equal_another(self):
        self.assertEqual(1,1)

    def test_not_equal(self):
        self.assertNotEqual(2,1)
    
    def test_true(self):
        self.assertTrue('a' in 'hello')
```

Получаем такой вывод:
```
$ python manage.py test
..F
======================================================================
FAIL: test_true (banners.tests.ExampleTestCase)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/code/banners/tests.py", line 11, in test_true
    self.assertTrue('a' in 'hello')
AssertionError: False is not true

----------------------------------------------------------------------
Ran 3 tests in 0.003s

FAILED (failures=1)
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
Destroying test database for alias 'default'...
ERROR: Job failed: exit code 1
```

9. Существует способ получить информацию о пройденных тестах более удобно. Это junit-файлы. Это xml файл с данными о каждом запущенном тесте и результате. Gitlab CI умеет парсить такие файлы и отображать в merge-request'ах подробную информацию.

Для python есть модуль xmlrunner, который позволяет такое делать.
Добавим его в requirements.txt

```
Django==2.1.7
mysqlclient==1.4.2
unittest-xml-reporting==2.4.0
```

А в banners/settings.py пропишем этот Runner тестов вместо стандартного:
```
TEST_RUNNER = 'xmlrunner.extra.djangotestrunner.XMLTestRunner'
TEST_OUTPUT_VERBOSE = True
TEST_OUTPUT_FILE_NAME = "unittest.xml"
TEST_OUTPUT_DIR = "/code/"

```

Получается:
```
$ python manage.py test
Creating test database for alias 'default'...
System check identified no issues (0 silenced).

Running tests...
----------------------------------------------------------------------
..F
======================================================================
FAIL [0.001s]: test_true (banners.tests.ExampleTestCase)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/code/banners/tests.py", line 11, in test_true
    self.assertTrue('a' in 'hello')
AssertionError: False is not true

----------------------------------------------------------------------
Ran 3 tests in 0.010s

FAILED (failures=1)

Generating XML reports...
Destroying test database for alias 'default'...
```
Вроде сработало. Осталось сказать gitlab ci где искать файл

10. Дописываем в gitlab-ci.yml

```
artifacts:
    reports:
      junit: /code/unittest.xml
```
