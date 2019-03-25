# Ansible

## 
На этот раз нам не нужна та огромная машина с gitlab, создадим несколько маленьких.

1. Создадим папку для Vagrantfile

```
iMac-Aleksej:develop kurt$ mkdir ansible_main
iMac-Aleksej:develop kurt$ cd ansible_main/
iMac-Aleksej:ansible_main kurt$ vi Vagrantfile
```

На этот раз воспользуемся возможностями Vagrant для provision машин.
```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.network "private_network", ip: "10.2.0.10"
  config.vm.provider "virtualbox" do |v|
    v.memory = 512
    v.cpus = 1
  config.vm.provision "shell",
    inline: "yum -y install epel-release && yum -y install ansible"
  end
end
```

```
iMac-Aleksej:ansible_main kurt$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'centos/7'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'centos/7' version '1901.01' is up to date...
==> default: Setting the name of the VM: ansible_main_default_1553422452303_80356

....


    default: Installed:
    default:   ansible.noarch 0:2.7.9-1.el7                                                  
    default: 
    default: Dependency Installed:
    default:   PyYAML.x86_64 0:3.10-11.el7                                                   
    default:   libtomcrypt.x86_64 0:1.17-26.el7                                              
    default:   libtommath.x86_64 0:0.42.0-6.el7                                              
    default:   libyaml.x86_64 0:0.1.4-11.el7_0                                               
    default:   python-babel.noarch 0:0.9.6-8.el7                                             
    default:   python-backports.x86_64 0:1.0-8.el7                                           
    default:   python-backports-ssl_match_hostname.noarch 0:3.5.0.1-1.el7                    
    default:   python-cffi.x86_64 0:1.6.0-5.el7                                              
    default:   python-enum34.noarch 0:1.0.4-1.el7                                            
    default:   python-httplib2.noarch 0:0.9.2-1.el7                                          
    default:   python-idna.noarch 0:2.4-1.el7                                                
    default:   python-ipaddress.noarch 0:1.0.16-2.el7                                        
    default:   python-jinja2.noarch 0:2.7.2-2.el7                                            
    default:   python-keyczar.noarch 0:0.71c-2.el7                                           
    default:   python-markupsafe.x86_64 0:0.11-10.el7                                        
    default:   python-paramiko.noarch 0:2.1.1-9.el7                                          
    default:   python-ply.noarch 0:3.4-11.el7                                                
    default:   python-pycparser.noarch 0:2.14-1.el7                                          
    default:   python-setuptools.noarch 0:0.9.8-7.el7                                        
    default:   python-six.noarch 0:1.9.0-2.el7                                               
    default:   python2-crypto.x86_64 0:2.6.1-15.el7                                          
    default:   python2-cryptography.x86_64 0:1.7.2-2.el7                                     
    default:   python2-jmespath.noarch 0:0.9.0-3.el7                                         
    default:   python2-pyasn1.noarch 0:0.1.9-7.el7                                           
    default:   sshpass.x86_64 0:1.06-2.el7                                                   
    default: 
    default: Complete!
```

Мы видим, что после поднятия машины запустился provision-скрипт, который установил ansible в эту машину. Можно провиженить уже запущенную машину по команде 
```
vagrant provision
```

2. Зайдем на машину и убедимся, что ansible работает

```
iMac-Aleksej:ansible_main kurt$ vagrant ssh
[vagrant@localhost ~]$ ansible --help
Usage: ansible <host-pattern> [options]

Define and run a single task 'playbook' against a set of hosts

...
```

3. Откроем еще одну консоль и создадим еще одну виртуалку, которую будем настраивать через ansible.

```
iMac-Aleksej:develop kurt$ mkdir ansible_slave
iMac-Aleksej:develop kurt$ cd ansible_slave/
iMac-Aleksej:ansible_slave kurt$ vi Vagrantfile
```

```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.network "private_network", ip: "10.2.0.11"
  config.vm.provider "virtualbox" do |v|
    v.memory = 256
    v.cpus = 1
  end
end
```

запустим её
```
vagrant up
```

4. Вернемся на машину с ansible и проверим, что из неё доступна эта свежесозданная машина по IP адресу

```
[vagrant@localhost ~]$ ping 10.2.0.11
PING 10.2.0.11 (10.2.0.11) 56(84) bytes of data.
64 bytes from 10.2.0.11: icmp_seq=1 ttl=64 time=0.643 ms
```

5. Создадим файл — список всех хостов, которые будут обслуживаться с помощью ansible.

```
[vagrant@st91 ~]$ mkdir ansible
[vagrant@st91 ~]$ cd ansible/
[vagrant@st91 ansible]$ vi inventory.yml
```

```
all:
  hosts:
    ansible_slave:
      ansible_host: 10.2.0.11
```


6. Теперь попробуем осуществить ping всех машин, прописанных в inventory файле

```
[vagrant@localhost ansible]$ ansible -i inventory.yml all -m ping
ansible_slave | UNREACHABLE! => {
    "changed": false, 
    "msg": "Failed to connect to the host via ssh: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).", 
    "unreachable": true
}
```

Как мы видим, авторизация не удалась.

7. Настроим авторизацию. Авторизация будет происходить по ssh-ключам. На машине с ansible нужно сгенерировать пару публичный-приватный ключи

```
[vagrant@localhost ansible]$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/vagrant/.ssh/id_rsa.
Your public key has been saved in /home/vagrant/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:kCeh0nLOJRfrLz+Vp7YTN0uxDc34RQdnCFD0xnRyfls vagrant@localhost.localdomain
The key's randomart image is:
+---[RSA 2048]----+
|      o   .++.+o=|
|   . . =     +.Bo|
|  o = B .    ++.E|
|   * = +    +.o =|
|    o . S  . * o |
|       .  + * o  |
|      . .. * o   |
|       o. + .    |
|        .o.o     |
+----[SHA256]-----+
```

Теперь публичный ключ (.ssh/id_rsa.pub) нужно положить на все машины, на которые нужно иметь доступ в файл .ssh/authorized_keys того же пользователя.
Ходить будем пользователем vargrant, который уже есть в системе.

8. На хост-машине в папке ansible_main сделаем следующее:

```
iMac-Aleksej:ansible_main kurt$ vagrant ssh-config
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/kurt/develop/ansible_main/.vagrant/machines/default/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL

```

В ответ vagrant сказал нам настройки, использовав которые, можно залогиниться на машину (он их использует в команде vagrant ssh)
Воспользуемся ими для передачи файла
```
iMac-Aleksej:ansible_main kurt$ scp -P 2222 -i /Users/kurt/develop/ansible_main/.vagrant/machines/default/virtualbox/private_key vagrant@127.0.0.1:/.ssh/id_rsa.pub .
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ECDSA key fingerprint is SHA256:DwdoCV7zmZpRqcwLjLjaTUWERWqyJ0+FAN9Hrtj1wXQ.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[127.0.0.1]:2222' (ECDSA) to the list of known hosts.
id_rsa.pub                                                                           100%  411   528.1KB/s   00:00    
```

Успех!

Теперь перейдем в папку ansible_slave и сделаем тоже самое, но уже загружая файл

```
iMac-Aleksej:ansible_slave kurt$ vagrant ssh-config
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2200
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/kurt/develop/ansible_slave/.vagrant/machines/default/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```

Копируем

```
iMac-Aleksej:ansible_slave kurt$ scp -P 2200 -i /Users/kurt/develop/ansible_slave/.vagrant/machines/default/virtualbox/private_key ../ansible_main/id_rsa.pub vagrant@127.0.0.1:/home/vagrant/
The authenticity of host '[127.0.0.1]:2200 ([127.0.0.1]:2200)' can't be established.
ECDSA key fingerprint is SHA256:oICdbo++Uf/dWOT8/zyMw+6lpPmMa5wDQF0hl0t8iOs.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[127.0.0.1]:2200' (ECDSA) to the list of known hosts.
id_rsa.pub                                                                           100%  411   453.0KB/s   00:00
```

Теперь на машине slave делаем
```
[vagrant@localhost ~]$ cat id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDqH/+bq5vX+ao36WhfAyvqeDOr25fKRiqlw7PS5DLlaP1oQGfYM2DiNWxMJ3vWfeb1mFpZH2RX781LKxaxc3Z3rqdwjz0W2aj8IYRkezC4AiHmpXdOaOp+Ig9f81kDOCCFFOWzROkpgU7/J4yhvfFAGmXDDlQRRDKE2/Avj6XzhDkVOTa9H94sfXfBQO3rbWrCoFX/V6jrJY0btMo0hulHEqOYF8jkYJsLSdtIRTPCLe/EZObZ/4OWWZmoggYbzDCTBIWvV9p894q9Tnb8e34yxZm/UTwKXTD/78nc6iQnrtte6elr1vL9cnBsYmMvoz+ycPTeXOQ6Yp0BT5h78KWL vagrant@localhost.localdomain
[vagrant@localhost ~]$ cat id_rsa.pub >> .ssh/authorized_keys
```

9. Проверяем с машины с ansible

```
[vagrant@localhost ansible]$ ansible -i inventory.yml all -m ping
ansible_slave | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

Попробуем запустить команду (посмотрим, под каким пользователем мы)

```
[vagrant@localhost ansible]$ ansible -i inventory.yml all -a "whoami"
ansible_slave | CHANGED | rc=0 >>
vagrant
```

Если мы хотим выполнять команду от root, нужно добавить --become, тогда сделается sudo в другого пользователя (по умолчанию root)

```
[vagrant@localhost ansible]$ ansible -i inventory.yml all -a "whoami" --become          
ansible_slave | CHANGED | rc=0 >>
root
```

10. Не хочется каждый раз прописывать файл с хостами и флаг --become (скорее всего настраивать машину будет от root). Поэтому заведем файл с конфигурацией:
```
[vagrant@localhost ansible]$ vi ansible.cfg
```

```
[defaults]
inventory=inventory.yml

[privilege_escalation]
become=True
```


Теперь можно делать просто вот так
```
[vagrant@localhost ansible]$ ansible all -a "whoami"
ansible_slave | CHANGED | rc=0 >>
root
```

11. Напишем playbook, который будет устанавливать docker на машину

```
[vagrant@localhost ansible]$ mkdir playbooks
[vagrant@localhost ansible]$ vi playbooks/setup_docker.yml
```

```
---
- hosts: all
  tasks:
    - name: installing dependencies
      yum:
        name:
          - yum-utils
          - device-mapper-persistent-data
          - lvm2
    - name: Add Docker GPG key
      rpm_key:
        key: https://download.docker.com/linux/centos/gpg
        state: present

    - name: Add Docker repository.
      get_url:
        url: "https://download.docker.com/linux/centos/docker-ce.repo"
        dest: '/etc/yum.repos.d/docker-ce.repo'

    - name: installing docker
      yum:
        name:
          - docker-ce

    - name: ensure docker is running
      service:
        name: docker
        state: started
        enabled: yes
```

запустим

```
[vagrant@localhost ansible]$ ansible-playbook playbooks/setup_docker.yml

PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [ansible_slave]

TASK [installing dependencies] *********************************************************************************************************************************************
changed: [ansible_slave]

TASK [Add Docker GPG key] **************************************************************************************************************************************************
changed: [ansible_slave]

TASK [Add Docker repository.] **********************************************************************************************************************************************
changed: [ansible_slave]

TASK [installing docker] ***************************************************************************************************************************************************
changed: [ansible_slave]

TASK [ensure docker is running] ********************************************************************************************************************************************
changed: [ansible_slave]

PLAY RECAP *****************************************************************************************************************************************************************
ansible_slave              : ok=6    changed=5    unreachable=0    failed=0   

```

попробуем запустить докер на машине slave:
```
[vagrant@localhost ~]$ sudo docker --version
Docker version 18.09.3, build 774a1f4
```

запустим плейбук еще раз
```
[vagrant@localhost ansible]$ ansible-playbook playbooks/setup_docker.yml 

PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [ansible_slave]

TASK [installing dependencies] *********************************************************************************************************************************************
ok: [ansible_slave]

TASK [Add Docker GPG key] **************************************************************************************************************************************************
ok: [ansible_slave]

TASK [Add Docker repository.] **********************************************************************************************************************************************
ok: [ansible_slave]

TASK [installing docker] ***************************************************************************************************************************************************
ok: [ansible_slave]

TASK [ensure docker is running] ********************************************************************************************************************************************
ok: [ansible_slave]

PLAY RECAP *****************************************************************************************************************************************************************
ansible_slave              : ok=6    changed=0    unreachable=0    failed=0   

```

Как мы видим, в этот раз плейбук ничего не изменил. Потому что всё уже было на месте.

## Тестирование ansible

С помощью ansible мы реализовываем концепцию "Инфраструктура как код". Мы можем полностью описать всю конфигурацию сервисов на серверах в коде и плейбуках. Теперь, поскольку это код, то код же можно тестировать. Для тестирования ansible-скриптов используется фреймворк "Molecule"

1. Тестирование происходит внутри docker, поэтому нам понадобится docker на машине с ansible. К счастью, у нас уже есть плейбука, которая его устанавливает. Осталось добавить в inventory локальный хост. Сделать это можно следующим образом:

inventory.yml
```
all:
  hosts:
    ansible_slave:
      ansible_host: 10.2.0.11
    localhost:
      ansible_connection: local
      ansible_host: 127.0.0.1
```

2. Запускаем и устанавливаем

```
[vagrant@st91 ansible]$ ansible-playbook playbooks/setup_docker.yml
```

```
[vagrant@st91 ansible]$ ansible-playbook playbooks/setup_docker.yml

PLAY [all] ********************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************
ok: [localhost]
ok: [ansible_slave]

TASK [installing dependencies] ************************************************************************************************************************************************
ok: [ansible_slave]
changed: [localhost]

TASK [Add Docker GPG key] *****************************************************************************************************************************************************
ok: [ansible_slave]
changed: [localhost]

TASK [Add Docker repository.] *************************************************************************************************************************************************
ok: [ansible_slave]
changed: [localhost]

TASK [installing docker] ******************************************************************************************************************************************************
ok: [ansible_slave]
changed: [localhost]

TASK [ensure docker is running] ***********************************************************************************************************************************************
ok: [ansible_slave]
changed: [localhost]

PLAY RECAP ********************************************************************************************************************************************************************
ansible_slave              : ok=6    changed=0    unreachable=0    failed=0
localhost                  : ok=6    changed=5    unreachable=0    failed=0

[vagrant@st91 ansible]$
```

Мы видим, что выполнение задач происходит последовательно, пока на каждом сервере не выполнится, следующая не запускается.

3. Установим molecule с помощью роли ansible
Пишем роль исходя из официальной инструкции https://molecule.readthedocs.io/en/latest/installation.html

```
[vagrant@st91 ansible]$ vi playbooks/setup_molecule.yml
```

```
---
- hosts: localhost
  tasks:
    - name: Add EPEL repository
      yum:
        name: epel-release

    - name: Installing dependencies
      yum:
        name:
          - gcc
          - python-pip
          - python-devel
          - openssl-devel
          - libselinux-python

    - name: Upgrade pip
      pip:
        name:
          - pip
          - setuptools
        extra_args: --upgrade

    - name: Install molecule
      pip:
        name: molecule
        extra_args:
          --ignore-installed PyYAML
```

4. Запускаем

```
[vagrant@st91 ansible]$ ansible-playbook playbooks/setup_molecule.yml

PLAY [localhost] **************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************
ok: [localhost]

TASK [Add EPEL repository] ****************************************************************************************************************************************************
ok: [localhost]

TASK [Installing dependencies] ************************************************************************************************************************************************
ok: [localhost]

TASK [Upgrade pip] ************************************************************************************************************************************************************
ok: [localhost]

TASK [Install molecule] *******************************************************************************************************************************************************
changed: [localhost]

PLAY RECAP ********************************************************************************************************************************************************************
localhost                  : ok=5    changed=1    unreachable=0    failed=0
```

5. Помимо тестирования, молекула помогает организовать код в виде "ролей". "Роль" — законченный набор инструкций по установке/настройке какого-то одного сервиса. Роли можно наследовать друг от друга. В итоговом playbook будет описан просто набор нужных ролей. Молекула тестирует каждую роль в отдельности. Организуем роль по установке докера с помощью molecule.

```
[vagrant@st91 ansible]$ mkdir roles
[vagrant@st91 ansible]$ cd roles
[vagrant@st91 roles]$ molecule init role -r docker
--> Initializing new role docker...
Initialized role in /home/vagrant/ansible/roles/docker successfully.
[vagrant@st91 roles]$ ls -lh docker/
total 4,0K
drwxrwxr-x. 2 vagrant vagrant   22 мар 25 08:56 defaults
drwxrwxr-x. 2 vagrant vagrant   22 мар 25 08:56 handlers
drwxrwxr-x. 2 vagrant vagrant   22 мар 25 08:56 meta
drwxrwxr-x. 3 vagrant vagrant   21 мар 25 08:56 molecule
-rw-r--r--. 1 vagrant vagrant 1,3K мар 25 08:56 README.md
drwxrwxr-x. 2 vagrant vagrant   22 мар 25 08:56 tasks
drwxrwxr-x. 2 vagrant vagrant   22 мар 25 08:56 vars
```

Описание того, за что отвечает каждая папка, можно найти в официальной документации по ansible: https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html

6. Посмотрим на файл с настройками молекулы. Для тестирования она будет поднимать docker-контейнер и в нем выполнять инструкции, описанные в роли. Dockerfile она генерирует самостоятельно, мы же можем управлять настройками, по которым она это будет делать.

```
[vagrant@st91 roles]$ cd docker/
[vagrant@st91 docker]$ cat molecule/default/molecule.yml
---
dependency:
  name: galaxy
driver:
  name: docker
lint:
  name: yamllint
platforms:
  - name: instance
    image: centos:7
provisioner:
  name: ansible
  lint:
    name: ansible-lint
verifier:
  name: testinfra
  lint:
    name: flake8
```

Поскольку мы собираемся устанавливать докер, получается, что он будет установлен внутри докер-контейнера, поэтому нам потребуется привелигированный режим. К тому же, мы будем устанавливать и включать службу, поэтому понадобится systemd.

[vagrant@st91 docker]$ vi molecule/default/molecule.yml
```
platforms:
  - name: instance
    image: centos/systemd:latest
    privileged: true
    command: /sbin/init
```

7. Попробуем поднять среду для тестов

```
[vagrant@st91 docker]$ molecule create
--> Validating schema /home/vagrant/ansible/roles/docker/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix

└── default
    ├── create
    └── prepare

--> Scenario: 'default'
--> Action: 'create'

    PLAY [Create] ******************************************************************

    TASK [Log into a Docker registry] **********************************************
    skipping: [localhost] => (item=None)

    TASK [Create Dockerfiles from image names] *************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Discover local Docker images] ********************************************
    failed: [localhost] (item=None) => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}
    fatal: [localhost]: FAILED! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}

    PLAY RECAP *********************************************************************
    localhost                  : ok=1    changed=1    unreachable=0    failed=1


ERROR:
```

Хм, что-то не получилось, а что — непонятно.
Чтобы увидеть более подробный лог нужно добавить --debug

```
[vagrant@st91 docker]$ molecule --debug create
--> Validating schema /home/vagrant/ansible/roles/docker/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix
...


       "msg": "Failed to import docker or docker-py - No module named docker. Try `pip install docker` or `pip install docker-py` (Python 2.6)"
    }

    PLAY RECAP *********************************************************************
    localhost                  : ok=1    changed=0    unreachable=0    failed=1


ERROR:
```

Не хватает docker-библиотеки для python. 

Допишем в плейбук по установке молекулы
playbooks/setup_molecule.yml
```
    - name: Install python docker
      pip:
        name:
          - docker
```

```
[vagrant@st91 ansible]$ ansible-playbook playbooks/setup_molecule.yml

PLAY [localhost] **************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************
ok: [localhost]

TASK [Add EPEL repository] ****************************************************************************************************************************************************
ok: [localhost]

TASK [Installing dependencies] ************************************************************************************************************************************************
ok: [localhost]

TASK [Upgrade pip] ************************************************************************************************************************************************************
ok: [localhost]

TASK [Install python docker] **************************************************************************************************************************************************
changed: [localhost]

TASK [Install molecule] *******************************************************************************************************************************************************
changed: [localhost]

PLAY RECAP ********************************************************************************************************************************************************************
localhost                  : ok=6    changed=1    unreachable=0    failed=0
```


Пробуем

```
[vagrant@st91 docker]$ molecule create
--> Validating schema /home/vagrant/ansible/roles/docker/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix

└── default
    ├── create
    └── prepare

--> Scenario: 'default'
--> Action: 'create'
Skipping, instances already created.
--> Scenario: 'default'
--> Action: 'prepare'
Skipping, prepare playbook not configured.
```

Успех!

После этого контейнер не убивается, он продолжает висеть

```
[vagrant@st91 docker]$ docker ps
CONTAINER ID        IMAGE                     COMMAND                  CREATED             STATUS              PORTS               NAMES
f2af967b1b18        molecule_local/centos:7   "bash -c 'while true…"   2 minutes ago       Up 2 minutes                            instance
[vagrant@st91 docker]$ molecule list
--> Validating schema /home/vagrant/ansible/roles/docker/molecule/default/molecule.yml.
Validation completed successfully.
Instance Name    Driver Name    Provisioner Name    Scenario Name    Created    Converged
---------------  -------------  ------------------  ---------------  ---------  -----------
instance         docker         ansible             default          true       false
```

8. Начнем перетаскивать таски из плейбука по установке докера в нашу роль.

```
[vagrant@st91 docker]$ vi tasks/main.yml
```

```
---

- name: installing dependencies
  yum:
    name:
      - yum-utils
      - device-mapper-persistent-data
      - lvm2
```

Запустить плейбук с этими задачами (плейбук лежит в molecule/default/playbook.yml) можно так:

```
[vagrant@st91 docker]$ molecule converge
--> Validating schema /home/vagrant/ansible/roles/docker/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix

└── default
    ├── dependency
    ├── create
    ├── prepare
    └── converge

--> Scenario: 'default'
--> Action: 'dependency'
Skipping, missing the requirements file.
--> Scenario: 'default'
--> Action: 'create'
Skipping, instances already created.
--> Scenario: 'default'
--> Action: 'prepare'
Skipping, prepare playbook not configured.
--> Scenario: 'default'
--> Action: 'converge'

    PLAY [Converge] ****************************************************************

    TASK [Gathering Facts] *********************************************************
    ok: [instance]

    TASK [docker : installing dependencies] ****************************************
    changed: [instance]

    PLAY RECAP *********************************************************************
    instance                   : ok=2    changed=1    unreachable=0    failed=0

```

Зайти и посмотреть, как всё прошло и что теперь на тестовой "машине" (а на самом деле в контейнере) можно так (не вводя всякие docker exec -ti)
```
molecule login
```

Мы попали в шелл внутри контейнера. Можно убедиться, что то, что мы ставили, установилось:
```
[root@instance /]# rpm -qa | grep lvm2
lvm2-libs-2.02.180-10.el7_6.3.x86_64
lvm2-2.02.180-10.el7_6.3.x86_64
```

Прибить контейнер и начать всё заново можно так:
```
molecule destroy
```

9. Полный цикл тестов можно запустить одной командой

```
[vagrant@st91 docker]$ molecule test
--> Validating schema /home/vagrant/ansible/roles/docker/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix

└── default
    ├── lint
    ├── cleanup
    ├── destroy
    ├── dependency
    ├── syntax
    ├── create
    ├── prepare
    ├── converge
    ├── idempotence
    ├── side_effect
    ├── verify
    ├── cleanup
    └── destroy

--> Scenario: 'default'
--> Action: 'lint'
--> Executing Yamllint on files found in /home/vagrant/ansible/roles/docker/...
Lint completed successfully.
--> Executing Flake8 on files found in /home/vagrant/ansible/roles/docker/molecule/default/tests/...
Lint completed successfully.
--> Executing Ansible Lint on /home/vagrant/ansible/roles/docker/molecule/default/playbook.yml...
    [701] Role info should contain platforms
    /home/vagrant/ansible/roles/docker/meta/main.yml:2
    {'meta/main.yml': {'__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'dependencies': [], u'galaxy_info': {u'description': u'your description', u'license': u'license (GPLv2, CC-BY, etc)', u'author': u'your name', u'company': u'your company (optional)', u'galaxy_tags': [], '__line__': 3, '__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'min_ansible_version': 1.2}, '__line__': 2}}

    [703] Should change default metadata: author
---
    /home/vagrant/ansible/roles/docker/meta/main.yml:2
    {'meta/main.yml': {'__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'dependencies': [], u'galaxy_info': {u'description': u'your description', u'license': u'license (GPLv2, CC-BY, etc)', u'author': u'your name', u'company': u'your company (optional)', u'galaxy_tags': [], '__line__': 3, '__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'min_ansible_version': 1.2}, '__line__': 2}}

    [703] Should change default metadata: description
    /home/vagrant/ansible/roles/docker/meta/main.yml:2
    {'meta/main.yml': {'__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'dependencies': [], u'galaxy_info': {u'description': u'your description', u'license': u'license (GPLv2, CC-BY, etc)', u'author': u'your name', u'company': u'your company (optional)', u'galaxy_tags': [], '__line__': 3, '__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'min_ansible_version': 1.2}, '__line__': 2}}

    [703] Should change default metadata: company
    /home/vagrant/ansible/roles/docker/meta/main.yml:2
    {'meta/main.yml': {'__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'dependencies': [], u'galaxy_info': {u'description': u'your description', u'license': u'license (GPLv2, CC-BY, etc)', u'author': u'your name', u'company': u'your company (optional)', u'galaxy_tags': [], '__line__': 3, '__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'min_ansible_version': 1.2}, '__line__': 2}}

    [703] Should change default metadata: license
    /home/vagrant/ansible/roles/docker/meta/main.yml:2
    {'meta/main.yml': {'__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'dependencies': [], u'galaxy_info': {u'description': u'your description', u'license': u'license (GPLv2, CC-BY, etc)', u'author': u'your name', u'company': u'your company (optional)', u'galaxy_tags': [], '__line__': 3, '__file__': u'/home/vagrant/ansible/roles/docker/meta/main.yml', u'min_ansible_version': 1.2}, '__line__': 2}}

An error occurred during the test sequence action: 'lint'. Cleaning up.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Destroy molecule instance(s)] ********************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Wait for instance(s) deletion to complete] *******************************
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Delete docker network(s)] ************************************************

    PLAY RECAP *********************************************************************
    localhost                  : ok=2    changed=1    unreachable=0    failed=0

```

Ansible-lint просит нас указать платформу, автора, описание, компанию и лицензию, по которой сделана наша роль. Можно отключать отдельный проверки следующим образом:

molecule/default/molecule.yml
```
provisioner:
  name: ansible
  lint:
    name: ansible-lint
    options:
      x: ["701", "703"]
```

Теперь
```
[vagrant@st91 docker]$ molecule test
--> Validating schema /home/vagrant/ansible/roles/docker/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix

└── default
    ├── lint
    ├── cleanup
    ├── destroy
    ├── dependency
    ├── syntax
    ├── create
    ├── prepare
    ├── converge
    ├── idempotence
    ├── side_effect
    ├── verify
    ├── cleanup
    └── destroy

--> Scenario: 'default'
--> Action: 'lint'
--> Executing Yamllint on files found in /home/vagrant/ansible/roles/docker/...
Lint completed successfully.
--> Executing Flake8 on files found in /home/vagrant/ansible/roles/docker/molecule/default/tests/...
Lint completed successfully.
--> Executing Ansible Lint on /home/vagrant/ansible/roles/docker/molecule/default/playbook.yml...
Lint completed successfully.
--> Scenario: 'default'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Destroy molecule instance(s)] ********************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Wait for instance(s) deletion to complete] *******************************
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Delete docker network(s)] ************************************************

    PLAY RECAP *********************************************************************
    localhost                  : ok=2    changed=1    unreachable=0    failed=0


--> Scenario: 'default'
--> Action: 'dependency'
Skipping, missing the requirements file.
--> Scenario: 'default'
--> Action: 'syntax'

    playbook: /home/vagrant/ansible/roles/docker/molecule/default/playbook.yml

--> Scenario: 'default'
--> Action: 'create'

    PLAY [Create] ******************************************************************

    TASK [Log into a Docker registry] **********************************************
    skipping: [localhost] => (item=None)

    TASK [Create Dockerfiles from image names] *************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Discover local Docker images] ********************************************
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Build an Ansible compatible image] ***************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Create docker network(s)] ************************************************

    TASK [Determine the CMD directives] ********************************************
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Create molecule instance(s)] *********************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Wait for instance(s) creation to complete] *******************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=6    changed=4    unreachable=0    failed=0


--> Scenario: 'default'
--> Action: 'prepare'
Skipping, prepare playbook not configured.
--> Scenario: 'default'
--> Action: 'converge'

    PLAY [Converge] ****************************************************************

    TASK [Gathering Facts] *********************************************************
    ok: [instance]

    TASK [docker : installing dependencies] ****************************************
    changed: [instance]

    PLAY RECAP *********************************************************************
    instance                   : ok=2    changed=1    unreachable=0    failed=0


--> Scenario: 'default'
--> Action: 'idempotence'
Idempotence completed successfully.
--> Scenario: 'default'
--> Action: 'side_effect'
Skipping, side effect playbook not configured.
--> Scenario: 'default'
--> Action: 'verify'
--> Executing Testinfra tests found in /home/vagrant/ansible/roles/docker/molecule/default/tests/...
    ============================= test session starts ==============================
    platform linux2 -- Python 2.7.5, pytest-4.3.1, py-1.8.0, pluggy-0.9.0
    rootdir: /home/vagrant/ansible/roles/docker/molecule/default, inifile:
    plugins: testinfra-1.19.0
collected 1 item

    tests/test_default.py .                                                  [100%]

    ========================== 1 passed in 15.88 seconds ===========================
Verifier completed successfully.
--> Scenario: 'default'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Destroy molecule instance(s)] ********************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Wait for instance(s) deletion to complete] *******************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Delete docker network(s)] ************************************************

    PLAY RECAP *********************************************************************
    localhost                  : ok=2    changed=2    unreachable=0    failed=0


```

10. Давайте заполним всю роль по установке докера и проверим, что она работает

tasks/main.yml
```
---

- name: installing dependencies
  yum:
    name:
      - yum-utils
      - device-mapper-persistent-data
      - lvm2

- name: Add Docker GPG key
  rpm_key:
    key: https://download.docker.com/linux/centos/gpg
    state: present

- name: Add Docker repository.
  get_url:
    url: "https://download.docker.com/linux/centos/docker-ce.repo"
    dest: '/etc/yum.repos.d/docker-ce.repo'

- name: installing docker
  yum:
    name:
      - docker-ce

- name: ensure docker is running
  service:
    name: docker
    state: started
    enabled: yes
```

Кроме того, напишем на неё тест! В качестве теста запустим "docker run hello-world"

molecule/default/tests/test_default.py
```
import os

import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']).get_hosts('all')


def test_docker_works(host):
    assert host.run('docker run hello-world').rc == 0
```

Запускаем
```
[vagrant@st91 docker]$ molecule test
--> Validating schema /home/vagrant/ansible/roles/docker/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix

└── default
    ├── lint
    ├── cleanup
    ├── destroy
    ├── dependency
    ├── syntax
    ├── create
    ├── prepare
    ├── converge
    ├── idempotence
    ├── side_effect
    ├── verify
    ├── cleanup
    └── destroy

--> Scenario: 'default'
--> Action: 'lint'
--> Executing Yamllint on files found in /home/vagrant/ansible/roles/docker/...
Lint completed successfully.
--> Executing Flake8 on files found in /home/vagrant/ansible/roles/docker/molecule/default/tests/...
Lint completed successfully.
--> Executing Ansible Lint on /home/vagrant/ansible/roles/docker/molecule/default/playbook.yml...
Lint completed successfully.
--> Scenario: 'default'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Destroy molecule instance(s)] ********************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Wait for instance(s) deletion to complete] *******************************
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Delete docker network(s)] ************************************************

    PLAY RECAP *********************************************************************
    localhost                  : ok=2    changed=1    unreachable=0    failed=0


--> Scenario: 'default'
--> Action: 'dependency'
Skipping, missing the requirements file.
--> Scenario: 'default'
--> Action: 'syntax'

    playbook: /home/vagrant/ansible/roles/docker/molecule/default/playbook.yml

--> Scenario: 'default'
--> Action: 'create'

    PLAY [Create] ******************************************************************

    TASK [Log into a Docker registry] **********************************************
    skipping: [localhost] => (item=None)

    TASK [Create Dockerfiles from image names] *************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Discover local Docker images] ********************************************
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Build an Ansible compatible image] ***************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Create docker network(s)] ************************************************

    TASK [Determine the CMD directives] ********************************************
    ok: [localhost] => (item=None)
    ok: [localhost]

    TASK [Create molecule instance(s)] *********************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Wait for instance(s) creation to complete] *******************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=6    changed=4    unreachable=0    failed=0


--> Scenario: 'default'
--> Action: 'prepare'
Skipping, prepare playbook not configured.
--> Scenario: 'default'
--> Action: 'converge'

    PLAY [Converge] ****************************************************************

    TASK [Gathering Facts] *********************************************************
    ok: [instance]

    TASK [docker : installing dependencies] ****************************************
    changed: [instance]

    TASK [docker : Add Docker GPG key] *********************************************
    changed: [instance]

    TASK [docker : Add Docker repository.] *****************************************
    changed: [instance]

    TASK [docker : installing docker] **********************************************
    changed: [instance]

    TASK [docker : ensure docker is running] ***************************************
    changed: [instance]

    PLAY RECAP *********************************************************************
    instance                   : ok=6    changed=5    unreachable=0    failed=0


--> Scenario: 'default'
--> Action: 'idempotence'
Idempotence completed successfully.
--> Scenario: 'default'
--> Action: 'side_effect'
Skipping, side effect playbook not configured.
--> Scenario: 'default'
--> Action: 'verify'
--> Executing Testinfra tests found in /home/vagrant/ansible/roles/docker/molecule/default/tests/...
    ============================= test session starts ==============================
    platform linux2 -- Python 2.7.5, pytest-4.3.1, py-1.8.0, pluggy-0.9.0
    rootdir: /home/vagrant/ansible/roles/docker/molecule/default, inifile:
    plugins: testinfra-1.19.0
collected 1 item

    tests/test_default.py .                                                  [100%]

    ========================== 1 passed in 16.68 seconds ===========================
Verifier completed successfully.
--> Scenario: 'default'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Destroy molecule instance(s)] ********************************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Wait for instance(s) deletion to complete] *******************************
    changed: [localhost] => (item=None)
    changed: [localhost]

    TASK [Delete docker network(s)] ************************************************

    PLAY RECAP *********************************************************************
    localhost                  : ok=2    changed=2    unreachable=0    failed=0
```
Всё прошло
