---
layout: post
title: "Как начать использовать Ansible и жить проще"
date: 2015-07-22 00:00:00 +0300
intro: >
    Вчера появилась задачка – добавить ssh ключ в authorized_keys для
    определенного пользователя на все свои серверы. И если фраза "все свои
    серверы" раньше меня особо не пугала, то сейчас, как минимум, напрягает.
    Пришлось начать осваивать еще один крутой инструмент.
categories: ru
tags: linux ansible
---

Честно говоря, я уже давно пытался раскурить Chef и Puppet для управления серверами. Но с наскока это сделать не удавалось. Недавно я ознакомился с Ansible и он подкупил меня своей простотой. Итак, теперь постараюсь без воды показать, как можно решить такую задачу, как добавление ssh ключа в authorized_keys на всех серверах. Причем, сначала нужно убедиться, что нужный нам пользователь существует.

Сначала установим Ansible. Для OS X это легко сделать через Homebrew:

```
brew install ansible
```

Первое, что нам нужно, это файл инвентаризации. Он описывает список всех ваших хостов и позволяет их разбивать на группы. Создадим этот файл и назовем его `hosts.ini`.

```
[frontend]
deploy@server01-fqdn.com
deploy@server02-fqdn.com
deploy@server03-fqdn.com
deploy@server04-fqdn.com

[backend]
deploy@server05-fqdn.com
deploy@server06-fqdn.com
deploy@server07-fqdn.com
deploy@server08-fqdn.com

[db]
deploy@server09-fqdn.com
deploy@server10-fqdn.com
```

Думаю, объяснять здесь что либо будет излишним.

Для начала вы можете запустить ansible в ad-hoc режиме. Например можно проверить, что все серверы доступны с помощью модуля ping.

```
ansible all -i hosts.ini -m ping
```

Здесь all обозначает все серверы из файла инвентаризации. Здесь мы через ключ -i передаем путь к нашему файлу инвентаризации, если этого не сделать, будет использован дефолтный файл, который лежит где-то в /etc. При желании мы можем заменить all на backend и у нас будет применено действие только к группе backend.

Пингануть серверы и убедиться, что все они живы и доступны по ssh – это конечно хорошо. Но давайте что-нибудь поинтереснее. Например, посмотрим аптайм всех этих серверов.

```
ansible all -i hosts.ini -m command -a 'uptime'
```

## Playbooks

Собственно, ad-hoc режим довольная полезная вещь. Однако, большинство задач подразумевает под собой выполнение какой-то цепочки действий. Для этого мы будем использовать так называемые playbooks, описываемые в YAML. Давайте напишем playbook, который решит поставленную проблему.

{% raw %}
```ruby
- name: Setup supervision support
  hosts: all
  tasks:
    - name: install libselinux-python
      yum: name=libselinux-python state=present
      sudo: yes

    - name: add user itm to system
      user: name=supervision shell=/bin/bash home=/home/supervision
      sudo: yes

    - name: add authorized key  
      authorized_key: user=supervision key="{{ lookup('file', '/Users/dlysenko/.ssh/supervision.pub') }}"
      sudo: yes
      sudo_user: supervision
```
{% endraw %}

И запустим его:

```
ansible-playbook supervision.yml -i hosts.ini
```

Итак, здесь сначала описывается название плейбука и указывается группа хостов, на которых он должен быть выполнен. Затем идет раздел tasks, который и включает в себя описания всех действий и состояний, которые должны быть достигнуты. В первом таске мы используем модуль yum и убеждаемся, что в системе установлен пакет libselinux-python, который необходим для корректной работы ansbile. Используем sudo, так как на сервер мы логинимся под юзером deploy (это указано в файле инвентаризации, хотя в playbook-е можно переопределить пользователя). Во втором таске мы используем модуль user и убеждаемся, что заданный пользователь создан в системе. Например, у меня на части серверов уже был создан нужный пользователь. Наконец, в третьем таске мы делаем то, ради чего все это затеяли. С помощью модуля authorized_key мы добавляем ключ, который находится на локальной машине по указанному пути. Причем делаем мы это через sudo от имени указанного пользователя, чтобы верно были выставлены права на файл.

Таким образом, потратив буквально пару минут на написание playbook я получил возможность раскидывать необходимый ключ на неограниченное количество серверов всего одной командой на локальной машине. И это только начало. Ведь вы можете написать такой playbook, который будет из чистой системы разворачивать вам полноценный веб-сервер. И после этого вы вообще не будете понимать, как раньше у вас хватало терпения настраивать сервера один за одним вручную.
