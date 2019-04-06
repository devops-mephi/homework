# Blue green deploy

На прошлом занятии мы остановились на схеме деплоя через ansible, которая обладает одним существенным недостатком: во время обновления сервис перестает работать и отдает ошибки 5xx.
Это неприемлимо.
Одним из способов решения данной проблемы (обновление без прекращения в обслуживании) является метод blue-green деплоя.
В чем его суть:
Одновременно на сервере развернуты две копии сервиса: "blue" и "green". Одна из них является продакшеном и на неё роутятся запросы пользователей. Далее при деплое обновляется другая копия. После этого проверяется, работает ли она и только после этого запросы перенаправляются на эту копию.

Попробуем реализовать такую схему.

1. Начнем с поднятия машин ansible_main и ansible_slave

2. Переместим пока tasks, что получились на прошлом занятии, чтобы начать всё заново

```
[vagrant@localhost ~]$ cd ansible/
[vagrant@localhost ansible]$ mv roles/banners/tasks/main.yml roles/banners/tasks/main.yml.old
```

3. Начнем писать новую последовательность действий, начав с определения и сохранения текущей продакшен-копии.

```
[vagrant@localhost ansible]$ vi roles/banners/tasks/main.yml
```

Будем хранить цвет текущей продакшен-копии в файле /etc/banners_color. Перед тем как что-то делать, нужно проверить, что файл такой вообще есть.

```
---
- name: Check that the /etc/banners_color exists
  stat:
    path: /etc/banners_color
  register: color_file_stat
```
Теперь в следующих задачах мы можем проверять, есть ли такой файл.

4. Если файла нет, то будем считать, что цвет - blue.

```
- name: If it does not exists then it is blue
  set_fact:
    production_color: blue
  when:
    color_file_stat.stat.exists == False
```

5. Если файл есть, прочитаем его. Один из способов: выполнить команду cat и записать её вывод в переменную.

```
- name: Reading /etc/banners_color to check for production color
  command: "cat /etc/banners_color"
  register: color_file
  when:
    color_file_stat.stat.exists == True

- name: Getting production color from file
  set_fact:
    production_color: "{{ color_file.stdout }}"
  when:
    color_file_stat.stat.exists == True
```

6. Дальше решаем, какую копию мы будем обновлять

```
- name: Deciding which copy we will update
  set_fact:
    update_color: "{% if production_color == 'blue' %}green{% else %}blue{% endif %}"
```

7. Вывести что-то в процессе исполнения плейбуки можно вот так
```
- debug:
    msg: "Current production is {{ production_color }}. Will update {{ update_color }}"
```

8. Ну и запишем в файл новую, обновленную копию. Это будет финалом нашей плейбуки, добавлять сам процесс обновления будем перед ним.

```
- name: Writing new production color to /etc/banners_color
  copy:
    dest: /etc/banners_color
    content: "{{ update_color }}"
```

9. Тестируем

```
[vagrant@localhost ansible]$ ansible-playbook playbooks/deploy_banners.yml --vault-id ansible_secret 

PLAY [ansible_slave] **************************************************************************

TASK [Gathering Facts] ************************************************************************
ok: [ansible_slave]

TASK [banners : Check that the /etc/banners_color exists] *************************************
ok: [ansible_slave]

TASK [banners : If it does not exists then it is blue] ****************************************
ok: [ansible_slave]

TASK [banners : Reading /etc/banners_color to check for production color] *********************
skipping: [ansible_slave]

TASK [banners : Getting production color from file] *******************************************
skipping: [ansible_slave]

TASK [banners : Deciding which copy we will update] *******************************************
ok: [ansible_slave]

TASK [banners : debug] ************************************************************************
ok: [ansible_slave] => {
    "msg": "Current production is blue. Will update green"
}

TASK [banners : Writing new production color to /etc/banners_color] ***************************
changed: [ansible_slave]

PLAY RECAP ************************************************************************************
ansible_slave              : ok=6    changed=1    unreachable=0    failed=0
```

Проверяем на ansible_slave:

```
[vagrant@localhost ~]$ cat /etc/banners_color 
green
```

Еще раз

```
[vagrant@localhost ansible]$ ansible-playbook playbooks/deploy_banners.yml --vault-id ansible_secret 

PLAY [ansible_slave] **************************************************************************

TASK [Gathering Facts] ************************************************************************
ok: [ansible_slave]

TASK [banners : Check that the /etc/banners_color exists] *************************************
ok: [ansible_slave]

TASK [banners : If it does not exists then it is blue] ****************************************
skipping: [ansible_slave]

TASK [banners : Reading /etc/banners_color to check for production color] *********************
changed: [ansible_slave]

TASK [banners : Getting production color from file] *******************************************
ok: [ansible_slave]

TASK [banners : Deciding which copy we will update] *******************************************
ok: [ansible_slave]

TASK [banners : debug] ************************************************************************
ok: [ansible_slave] => {
    "msg": "Current production is green. Will update blue"
}

TASK [banners : Writing new production color to /etc/banners_color] ***************************
changed: [ansible_slave]

PLAY RECAP ************************************************************************************
ansible_slave              : ok=7    changed=2    unreachable=0    failed=0
```

Теперь
```
[vagrant@localhost ~]$ cat /etc/banners_color 
blue
```

Позапускайте еще, убедитесь, что цвет копии успешно меняется туда-сюда

10. Добавим поднятия контейнера banners, скопировав задачи из main.yml.old + поднятие сети. Но, поменяем только имя и алиас контейнера, добавив _blue или _green в конец:

```
- name: Create a network
  docker_network:
    name: network_docker

- name: Create banners container
  docker_container:
    name: "banners_web_{{ update_color }}"
    image: devopsmephi/banners:latest
    pull: yes
    recreate: yes
    networks:
      - name: network_docker
        aliases:
          - "banners_web_{{ update_color }}"
    volumes:
      - "/etc/banners.conf.py:/code/banners/local_settings.py"
```

Позапускайте роль, убедитесь, что каждый раз переподнимается нужный контейнер:

```
[vagrant@localhost ~]$ sudo docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS               NAMES
4ba8780b81c2        devopsmephi/banners:latest   "/bin/sh -c 'python …"   6 seconds ago       Up 3 seconds                            banners_web_green
9e30091751f4        devopsmephi/banners:latest   "/bin/sh -c 'python …"   40 seconds ago      Up 35 seconds                           banners_web_blue
```
