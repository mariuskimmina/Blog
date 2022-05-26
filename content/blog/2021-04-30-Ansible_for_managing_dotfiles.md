---
title: Managing my system configurations with Ansible
author: Marius Kimmina
date: 2021-05-05 14:10:00 +0800
categories: [DevOps, Ansible]
published: false
tags: [Ansible, Dotfiles]
---


## 1. Introduction
A common problem for a lot of Developers, Sysadmins and the like is that they have a lot of custom configurations
on they're system. These configurations can be a lot of different things, aliases for commands or shortcuts for example. 
People grow used to they're configurations and they develop an efficent workflow with them. The problem with this is, 
that it becomes harder to work with off the shelf systems. Especially for people that do a lot of they're work in
VMs or otherwise virtualized environments, they find themselves missing they're cosy configurations quite often. 
They are then faced with 2 options:

* Install all they're custom software and configurations on the system
* Work with the system as is

The first of these solutions takes a lot of time upfront, the second one might be fine for short tasks but if they are 
supposed to work with the given system for a long time, this becomes inefficient. This is were Ansible comes in, it can be used 
to reduce the setup time of a new system dramatically. In fact, once all the ansible playbook is ready, new systems can be 
configured with one single command, leaving the developer/admin free to do other things while his system is getting ready.

## 2. What is Ansible?

Well, from the Ansible website itself: "Ansible is a universal language, unraveling the mystery of how work gets done. 
Turn tough tasks into repeatable playbooks. Roll out enterprise-wide protocols with the push of a button." 
With ansible we define a job that needs to get done with YAML files, these so called `playbooks` can then be used over over again.

### 2.1 Playbooks
Playbooks are on the highest level of the ansible hierarchy, a `playbook` defines a tasks that is in itself complete and does not depend
on any other tasks. A `playbook` can be made up of one big .yml file or it can broken up into multiple `roles`. 

### 2.2 Roles

### 2.3 Tasks

### Example Role: Tmux
So lets, look at an example, installing and configuring Neovim with ansible. My Tmux role has the following layout:

```
├── files
│   └── .tmux.conf
└── tasks
    ├── darwin.yml
    ├── debian.yml
    ├── main.yml
    └── redhat.yml
```

Taking a closer look at `tasks/main.yml` we can see that we first start by importing tasking from `redhat.yml`, `debian.yml`
and `darwin.yml` depending on the os/distro the playbook is running on.

```
- import_tasks: redhat.yml
  when: ansible_os_family == "RedHat"
- import_tasks: debian.yml
  when: ansible_os_family == "Debian"
- import_tasks: darwin.yml
  when: ansible_os_family == "Darwin"

- name: Check if .tmux.conf already exists
  stat: path="{{dotfiles_user_home}}/.tmux.conf"
  register: tmux_stat

- name: Back up tmux.conf if it exists
  command: mv ~/.tmux.conf ~/.tmux.conf.bak
  args:
    creates: "{{dotfiles_user_home}}/.tmux.conf.bak"
  when: tmux_stat.stat.exists

- name: Symlink tmux.conf
  file:
    src: "{{ dotfiles_home }}/roles/tmux/files/.tmux.conf"
    dest: "{{dotfiles_user_home}}/.tmux.conf"
    state: link

- name: Install Tmux Plugin Manger (tpm)
  git:
    repo: https://github.com/tmux-plugins/tpm
    dest: ~/.tmux/plugins/tpm
    version: master
```

To make sure that existing configurations are not overwritten by accident there is a check if .tmux.conf already exists 
in the users $HOME directory. If it does exist, it is renamed to "tmux.conf.bak", this way it's content is not delted by accident.
After that we create a symlink from the `.tmux.conf` in our ansible playrole to the home directory of the user.
Using a symlink here has the clear benefit that all changes between the original file and the symlink are synchronized. 
That means that if the user changes his `.tmux.conf` in his home directory, the changes are carried over to the repository 
immediately and can be pushed upstream.


### 3. Why Ansible for managing system configurations?

I have done some research on how other people handled this problem of managing their system configurations and most of the time
people were using some sort of bash script to place all the config files where they belong, but this approach doesn't work as well 
when you are trying to support different systems such as Debian based Linux distros, Redhat based Linux distros as well as maybe MacOS.
Also overtime your system configuration is going to grow and ansible makes it really easy to keep everything organized and in it's own place.
Every program that I want to configure has it's own role, all the associated configuration files go under files/ in the role of the program.
So ease of maintainability was really the main reason for me to choose ansible for this job instead of writing bash scripts like I saw many other
people do.


