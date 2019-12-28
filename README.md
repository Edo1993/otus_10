# otus_10
# Автоматизация администрирования. Ansible

Подготовить стенд на Vagrant как минимум с одним сервером. На сервере используя Ansible необходимо развернуть nginx со следующими условиями:
- использовать модуль yum/apt
- конфигурационные файлы должны быть взяты из шаблона jinja2 с переменными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible
* Сделать все это с использованием Ansible роли

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
mkdir 10_otus
cd 10_otus
mkdir 1
mkdir 2
cp Vagrantfile 10_otus/1/
cp Vagrantfile 10_otus/2/
```
Поднимаем vm командой ```vagrant_up```, для подключения к хосту nginx нам необходимо будет передать множество параметров - это особенность Vagrant. Узнать эти параметры можно с помощью ```vagrant ssh-config```
В ответ получаем:

![Image alt](https://github.com/Edo1993/otus_10/raw/master/101.png)
![Image alt](https://github.com/Edo1993/otus_13/raw/master/102.png)
