# otus_10
# Автоматизация администрирования. Ansible

Подготовить стенд на Vagrant как минимум с одним сервером. На сервере используя Ansible необходимо развернуть nginx со следующими условиями:
- использовать модуль yum/apt
- конфигурационные файлы должны быть взяты из шаблона jinja2 с переменными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible

клонировать [отсюда](https://github.com/Edo1993/otus_10/tree/master/1_part), выполнить
```
ansible-playbook nginx.yml
```

* Сделать все это с использованием Ansible роли
клонировать [отсюда](https://github.com/Edo1993/otus_10/tree/master/2_part), выполнить
```
ansible-playbook nginx.yml
```

# Установка Ansible
(сначала шпарим на ubuntu)

Ansible требует для своей работы python. Проверить версию python:
```
python -V
```
т.к у меня его нет, устанавливаем:
```
sudo apt install python3
```
Далее переходим к установке Ansible [инструкция](https://docs.ansible.com/ansible/2.7/installation_guide/intro_installation.html#basics-what-will-be-installed)
```
sudo apt-get update
sudo apt-get install software-properties-common -y
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt-get install ansible -y
```
Убедиться, что ansible установлен корректно:
```
ansible --version
```
# Подготовка окружения
Создать каталог Ansible, положить в него этот [Vagrantfile](https://gist.github.com/lalbrekht/f811ce9a921570b1d95e07a7dbebeb1e)
Хочу, чтобы были подняты 2 машины - для более наглядного использования переменных в дальнейшем, внутри каталога создам 2 папки (1, 2) под vagrantfile, указанный выше, в одном из них изменю имя, чтобы не путались.
```
mkdir 10
cd 10
mkdir 1
mkdir 2
cp Vagrantfile 10/1/
cp Vagrantfile 10/2/
```
Поднимаем vm командой ```vagrant_up```, для подключения к хосту nginx нам необходимо будет передать множество параметров - это особенность Vagrant. Узнать эти параметры можно с помощью ```vagrant ssh-config```
В ответ получаем:

![Image alt](https://github.com/Edo1993/otus_10/raw/master/101.png)

![Image alt](https://github.com/Edo1993/otus_10/raw/master/102.png)

Создадим отедльную директорию под inventory-файл, и сам файл hosts.
```
mkdir staging
cd staging
vi hosts
```
Содержимое файла hosts
```
[web]
nginx2 ansible_host=127.0.0.1 ansible_port=2201 ansible_private_key_file=/home/ed/otus/10/2/.vagrant/machines/nginx2/virtualbox/private_key
nginx ansible_host=127.0.0.1 ansible_port=2200 ansible_private_key_file=/home/ed/otus/10/1/.vagrant/machines/nginx/virtualbox/private_key
```
Отредактируем файл ```ansible.cfg```

inventory - путь к моему hosts

remote_user = vagrant - эту строку можно не указывать, но тогда в hosts для каждой vm должен быть указан ```ansible_user=vagrant```

host_key_checking = False - отменить проверку пароля
```
[defaults]
inventory = /home/ed/otus/10/staging/hosts
remote_user = vagrantinventory
host_key_checking = False
retry_files_enabled = False
```
# Ansible

После этого убедимся, что Ansible может управлять нашим хостом. Сделать это
можно с помощью команды:
```
ansible -i /home/ed/otus/10/staging/hosts all -m ping
```
![Image alt](https://github.com/Edo1993/otus_10/raw/master/103.png)

Теперь, когда убедились, что у нас все подготовлено - установлен Ansible, поднят хост для теста и Ansible имеет к нему доступ, мы можем конфигурировать наш хост. Для начала воспользуемся *Ad-Hoc командами* и выполним некоторые удаленные команды на нашем хосте:
```
ansible -i /home/ed/otus/10/staging/hosts all -m command -a "uname -r"
```
![Image alt](https://github.com/Edo1993/otus_10/raw/master/104.png)
```
ansible -i /home/ed/otus/10/staging/hosts all -m systemd -a name=firewalld
```
![Image alt](https://github.com/Edo1993/otus_10/raw/master/105.png)
Вывод большой, нас интересует только строка
```
"status": {
        ...
        "ActiveState": "inactive", 
        ...
```
Установим пакет epel-release на хосты
```
ansible -i /home/ed/otus/10/staging/hosts all -m yum -a "name=epel-release state=present" -b
```
![Image alt](https://github.com/Edo1993/otus_10/raw/master/106.png)

Напишем простой Playbook, который будет устанавливать пакет epel-release. Создадим файл epel.yml со следующим содержимым
```inventory
---
- name: Install EPEL Repo
  hosts: all
  become: true
  tasks:
   - name: Install EPEL Repo package from standard repo
     yum:
      name: epel-release
      state: present
```
Запустим Playbook:
```
ansible-playbook epel.yml
```
![Image alt](https://github.com/Edo1993/otus_10/raw/master/107.png)

Затем выполним команду ```ansible -i /home/ed/otus/10/staging/hosts all -m yum -a "name=epel-release state=absent" -b```, ещё раз запустим Playbook, разница в выводе:
![Image alt](https://github.com/Edo1993/otus_10/raw/master/108.png)

/во втором случае устанавливалось подольше?/

# Homework

За основу будущего playbook возьмем уже созданный нами файл epel.yml, переименовав его в nginx.yml. Добавим в него установку пакета NGINX. Измененный файл будет выглядеть так:
```
---
- name: NGINX | Install and configure NGINX
  hosts: all
  become: true
  
  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages
```
В файл добавили Tags. Теперь можно вывести в консоль список тегов и выполнить, например, только установку NGINX. В нашем случае так, например, можно осуществлять его обновление. Выведем в консоль все теги:
```
ansible-playbook nginx.yml --list-tags
```
![Image alt](https://github.com/Edo1993/otus_10/raw/master/109.png)
Запустим толþко установку NGINX:
```
ansible-playbook nginx.yml -t nginx-package
```
![Image alt](https://github.com/Edo1993/otus_10/raw/master/110.png)
Далее добавим шаблон для конфига NGINX и модуль, который будет копировать этот шаблон на хост, пропишем в Playbook необходимую нам переменную для того, чтобы NGINX слушал на порту 8080. На данном этапе Playbook будет выглядеть так:
```
---
- name: NGINX | Install and configure NGINX
  hosts: all
  become: trueinventory
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags:
        - nginx-configuration
```
Теперь создадим handler и добавим notify к копированию шаблона. Теперь каждый раз когда конфиг будет изменяться - сервис перезагрузиться. Результирующий файл:
```
---
- name: NGINX | Install and configure NGINX
  hosts: all
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes
    
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```
Запускаем измененный playbook.

(мой скрин не совсем корректный - первый vagrantfile поднялся сразу, второй нет, пришлось поправлять, но переделывать до идеального скрина - не хватает терпения)
![Image alt](https://github.com/Edo1993/otus_10/raw/master/111.png)
проверяем, что работает

http://192.168.11.150:8080/
![Image alt](https://github.com/Edo1993/otus_10/raw/master/112.png)
http://192.168.11.151:8080/
![Image alt](https://github.com/Edo1993/otus_10/raw/master/113.png)
Все файлы, по которым можно воспроизвести - [тут](https://github.com/Edo1993/otus_10/tree/master/1_part)

# Роль

Для создания роли
```
ansible-galaxy init nginx
```
Получим подобный каталог папок и файлов (лишнее потом удалим)
![Image alt](https://github.com/Edo1993/otus_10/raw/master/114.png)

Распиливаем наш playbook, перемещая всё по соответствующим директориям.
 - 1 В раздел handlers - перемещаем всё из блока handlers в playbook (самом playbook перенесёное выпиливаем - справедливо и для всего нижеследующего).
Итого, ```/handlers/main.yml``` получит следующую начинку:
```
---
# handlers file for nginx
- name: restart nginx
  systemd:
    name: nginx
    state: restarted
    enabled: yes
    
- name: reload nginx
  systemd:
    name: nginx
    state: reloaded
```
 - 2 В раздел tasks - перемещаем всё из блока tasks в playbook.
```/tasks/main.yml``` :
```
---
# tasks file for nginx
- name: NGINX | Install EPEL Repo package from standart repo
  yum:
    name: epel-release
    state: present
  tags:
    - epel-package
    - packages

- name: NGINX | Install NGINX package from EPEL Repo
  yum:
    name: nginx
    state: latest
  notify:
    - restart nginx
  tags:
    - nginx-package
    - packages

- name: NGINX | Create NGINX config file from template
  template:
    src: templates/nginx.c---
# tasks file for nginx
- name: NGINX | Install EPEL Repo package from standart repo
  yum:
    name: epel-release
    state: present
  tags:
    - epel-package
    - packages

- name: NGINX | Install NGINX package from EPEL Repo
  yum:
    name: nginx
    state: latest
  notify:
    - restart nginx
  tags:
    - nginx-package
    - packages

- name: NGINX | Create NGINX config file from template
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify:
    - reload nginx
  tags:
    - nginx-configurationonf.j2
    dest: /etc/nginx/nginx.conf
  notify:
    - reload nginx
  tags:
    - nginx-configuration
```
 - 3 В раздел ```templates``` перенесла содержимое текущего ```templates```, выпилив существующую директорию.
 - 4 Раздел ```vars```
```
---
# vars file for nginx
nginx_listen_port: 8080
```

Остальные директории были удалены. Playbook принял следующий вид :
```
---
- name: NGINX | Install and configure NGINX
  hosts: all
  become: true

  roles:
    - nginx
```

(Если честно - то после создания каталога ролей я пошла спать, и на следующий день у меня нихрена не работало. Провозившись 2 часа стало понятно, что после рестарта вм изменили порты. Вылечилось простым изменением портов в inventory-файле. Для проверки - позвать ```vagrant ssh-config``` и проверить, что вывод совпадает с содержимым inventory)

Каталог с ролями должен лежать в той же директории, что и playbook - иначе у меня не стартануло.

Запускаем playbook, чтобы проверить, что всё работает
![Image alt](https://github.com/Edo1993/otus_10/raw/master/115.png)

Итого - всё это хозяйство с ролью лежит [здесь](https://github.com/Edo1993/otus_10/tree/master/2_part)
