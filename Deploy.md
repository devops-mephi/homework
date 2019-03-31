# Deploy сервиса в продакшен

## Docker registry

Существующие сборки в Gitlab CI, которые мы делали на прошлых занятиях, собирают и хранят образ локально. Для деплоя нужно перенести собранный образ на продакшен-машину. Это делают с помощью docker registry.
Мы не будем поднимать свой собственный, а воспользуемся официальным Docker Hub.

1. Зарегистрируйтесь на https://hub.docker.com/signup

2. После успешной регистрации создайте репозиторий, который будет называться banners. В итоге вы сможете пушить образ в ИМЯ_ПОЛЬЗОВАТЕЛЯ/banners.
В нашем примере это будет devopsmephi/banners

3. Нужно сделать docker login на машине с gitlab-ci раннерами. Они у нас установлены в машине с Gitlab CI.

```
[vagrant@localhost ~]$ sudo su gitlab-runner

[gitlab-runner@localhost vagrant]$ sudo docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: devopsmephi
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

```

4. Добавим в .gitlab ci третий стейдж "deploy"

```
stages:
  - build
  - test
  - deploy
```

и сборку, которая пока будет просто вешать тег latest и пушить в репозиторий. Мы будем вешать еще один тег, с хешом коммита в git, чтобы потом было легче расследовать проблемы.

```
deploy_to_prod:
  stage: deploy
  tags:
    - vagrant
  script:
    - docker tag banners_web:$CI_BUILD_REF_NAME devopsmephi/banners:$CI_COMMIT_SHORT_SHA
    - docker tag banners_web:$CI_BUILD_REF_NAME devopsmephi/banners:latest
    - docker push devopsmephi/banners:$CI_COMMIT_SHORT_SHA
    - docker push devopsmephi/banners:latest

  when: manual
```

5. Запустим её


```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on vagrant wC8Ldt2b
Using Shell executor...
Running on localhost.localdomain...
Fetching changes...
HEAD is now at 1fbc7d9 Update .gitlab-ci.yml
Checking out 1fbc7d97 as master...
Skipping Git submodules setup
$ docker tag banners_web:$CI_BUILD_REF_NAME devopsmephi/banners:$CI_COMMIT_SHORT_SHA
$ docker tag banners_web:$CI_BUILD_REF_NAME devopsmephi/banners:latest
$ docker push devopsmephi/banners:$CI_COMMIT_SHORT_SHA
The push refers to repository [docker.io/devopsmephi/banners]
5f737ef5aac8: Preparing
877c3b8b4986: Preparing
f4d7791fcb9d: Preparing
f61166e33473: Preparing
bb839e9783c7: Preparing
237ce60325c6: Preparing
1b976700da1f: Preparing
bde41e1d0643: Preparing
7de462056991: Preparing
3443d6cf0f1f: Preparing
237ce60325c6: Waiting
1b976700da1f: Waiting
bde41e1d0643: Waiting
7de462056991: Waiting
f3a38968d075: Preparing
a327787b3c73: Preparing
5bb0785f2eee: Preparing
3443d6cf0f1f: Waiting
f3a38968d075: Waiting
a327787b3c73: Waiting
bb839e9783c7: Mounted from library/python
5f737ef5aac8: Pushed
237ce60325c6: Mounted from library/python
f61166e33473: Pushed
f4d7791fcb9d: Pushed
7de462056991: Mounted from library/python
1b976700da1f: Mounted from library/python
3443d6cf0f1f: Mounted from library/python
bde41e1d0643: Mounted from library/python
f3a38968d075: Mounted from library/python
a327787b3c73: Mounted from library/python
5bb0785f2eee: Mounted from library/python
877c3b8b4986: Pushed
1fbc7d97: digest: sha256:2ee907f50abedadb8a5e5975eebd265d2ad8fb1c5ff604ca333e876bdae014d6 size: 3052
$ docker push devopsmephi/banners:latest
The push refers to repository [docker.io/devopsmephi/banners]
5f737ef5aac8: Preparing
877c3b8b4986: Preparing
f4d7791fcb9d: Preparing
f61166e33473: Preparing
bb839e9783c7: Preparing
237ce60325c6: Preparing
1b976700da1f: Preparing
bde41e1d0643: Preparing
7de462056991: Preparing
3443d6cf0f1f: Preparing
f3a38968d075: Preparing
a327787b3c73: Preparing
5bb0785f2eee: Preparing
bde41e1d0643: Waiting
7de462056991: Waiting
3443d6cf0f1f: Waiting
f3a38968d075: Waiting
a327787b3c73: Waiting
5bb0785f2eee: Waiting
237ce60325c6: Waiting
1b976700da1f: Waiting
f61166e33473: Layer already exists
877c3b8b4986: Layer already exists
5f737ef5aac8: Layer already exists
f4d7791fcb9d: Layer already exists
bb839e9783c7: Layer already exists
1b976700da1f: Layer already exists
237ce60325c6: Layer already exists
7de462056991: Layer already exists
bde41e1d0643: Layer already exists
3443d6cf0f1f: Layer already exists
a327787b3c73: Layer already exists
5bb0785f2eee: Layer already exists
f3a38968d075: Layer already exists
latest: digest: sha256:2ee907f50abedadb8a5e5975eebd265d2ad8fb1c5ff604ca333e876bdae014d6 size: 3052
Job succeeded
```

Как мы видим, сборка успешно запушила образ с тегом 1fbc7d97 и latest

## Установка mysql на конечную машину

Будем деплоить сервис на машину ansible_slave, созданную на прошлом занятии.
1. Погасим машину с gitlab ci, чтобы не отнимала ресурсы.
Добавим памяти машине ansible_slave в Vagrantfile

```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.network "private_network", ip: "10.2.0.11"
  config.vm.provider "virtualbox" do |v|
    v.memory = 512
    v.cpus = 1
  end
end
```

2. Запустим, зайдем на машину ansible_main и создадим новую роль для установки mysql

```
---
- hosts: ansible_slave
  tasks:
    - name: checking if MySQL Repo exists
      stat:
        path: /etc/yum.repos.d/mysql-community.repo
      register: mysql_repo_exists

    - name: Download MySQL Repo
      get_url:
        url: https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
        dest: /tmp
      when: mysql_repo_exists.stat.exists == false

    - name: Install MySQL Repo
      command: rpm -ivh /tmp/mysql57-community-release-el7-9.noarch.rpm
      when: mysql_repo_exists.stat.exists == false

    - name: Install MySQL
      yum:
        name:
          - mysql-server

    - name: Ensure MySQL is started and enabled
      service:
        name: mysqld
        enabled: yes
        state: started
```

Запустим
```
[vagrant@localhost ansible]$ ansible-playbook playbooks/setup_mysql.yml 

PLAY [ansible_slave] ***************************************************************************************

TASK [Gathering Facts] *************************************************************************************
ok: [ansible_slave]

TASK [Download MySQL Repo] *********************************************************************************
ok: [ansible_slave]

TASK [Install MySQL Repo] **********************************************************************************
 [WARNING]: Consider using the yum, dnf or zypper module rather than running 'rpm'.  If you need to use
command because yum, dnf or zypper is insufficient you can add 'warn: false' to this command task or set
'command_warnings=False' in ansible.cfg to get rid of this message.

changed: [ansible_slave]

TASK [Install MySQL] ***************************************************************************************
changed: [ansible_slave]

TASK [Ensure MySQL is started and enabled] *****************************************************************
changed: [ansible_slave]

PLAY RECAP *************************************************************************************************
ansible_slave              : ok=4    changed=1    unreachable=0    failed=0 
```

3. Теперь зайдем на машину ansible_slave и продолжим установку руками.

Во-первых нужно узнать какой пароль root пользователю был выставлен при установке.
Временный пароль был написан в логе, найдем его
```
[vagrant@localhost ~]$ sudo grep 'temporary password' /var/log/mysqld.log
2019-03-31T16:45:41.966295Z 1 [Note] A temporary password is generated for root@localhost: uwOb2<YXMi3X
```

4. Далее запустим скрипт, который "засекьюрит" инсталляцию mysql (запретит логин root-пользователя извне, логин анонимных пользователей, удалит тестовую БД)

```
sudo mysql_secure_installation
```

```
[vagrant@localhost ~]$ sudo mysql_secure_installation

Securing the MySQL server deployment.

Enter password for user root: 

The existing password for the user account root has expired. Please set a new password.

New password: 

Re-enter new password: 
 ... Failed! Error: Your password does not satisfy the current policy requirements

New password: 

Re-enter new password: 
The 'validate_password' plugin is installed on the server.
The subsequent steps will run with the existing configuration
of the plugin.
Using existing password for root.

Estimated strength of the password: 100 
Change the password for root ? ((Press y|Y for Yes, any other key for No) : y

New password: 

Re-enter new password: 

Estimated strength of the password: 100 
Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) : y
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.


Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y
Success.

By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.


Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y
 - Dropping test database...
Success.

 - Removing privileges on test database...
Success.

Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
Success.

All done! 
```

5. Теперь выполним первоначальную настройку БД.

Нам нужно:
создать новую бд mephi
создать нового пользователя developer с правами на бд mephi
создать таблицы и наполнить их из init.sql файла

Для начала скачаем init.sql в текущую папку

```
[vagrant@localhost ~]$ wget https://raw.githubusercontent.com/devops-mephi/banners/master/db/init.sql
--2019-03-31 17:04:27--  https://raw.githubusercontent.com/devops-mephi/banners/master/db/init.sql
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 151.101.84.133
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|151.101.84.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 13812 (13K) [text/plain]
Saving to: 'init.sql'

100%[==================================================================>] 13,812      --.-K/s   in 0.02s   

2019-03-31 17:04:27 (610 KB/s) - 'init.sql' saved [13812/13812]
```

Теперь запустим консоль к mysql используя установленный выше пароль root
```
[vagrant@localhost ~]$ mysql -uroot -p 
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 5.7.25 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

6. Создаем БД

```
mysql> create database mephi;
Query OK, 1 row affected (0.00 sec)
```

7. Создаем пользователя developer с паролем и доступом к бд mephi

```
mysql> create user 'developer'@'localhost' identified by 'Developerpa$sw0rd';
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on mephi.* to 'developer'@'localhost';
Query OK, 0 rows affected (0.00 sec)
```

8. Наполняем бд mephi

```
mysql> use mephi
Database changed
mysql> source init.sql
Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

.....

```

9. Пробуем подключиться как developer

```
[vagrant@localhost ~]$ mysql -udeveloper -p mephi
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 17
Server version: 5.7.25 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show tables;
+----------------------------+
| Tables_in_mephi            |
+----------------------------+
| auth_group                 |
| auth_group_permissions     |
| auth_permission            |
| auth_user                  |
| auth_user_groups           |
| auth_user_user_permissions |
| django_admin_log           |
| django_content_type        |
| django_migrations          |
| django_session             |
+----------------------------+
10 rows in set (0.00 sec)

```

