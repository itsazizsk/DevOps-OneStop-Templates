# 1. ANSIBLE — COMPLETE SAMPLES
### 1.1 Inventory File (hosts.ini)
#### `INI Format`
```
[web]
web1 ansible_host=10.0.1.10
web2 ansible_host=10.0.1.11

[db]
db1 ansible_host=10.0.2.20

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```
#### `YAML Inventory`
```
all:
  vars:
    ansible_user: ubuntu
  children:
    web:
      hosts:
        web1:
          ansible_host: 10.0.1.10
        web2:
          ansible_host: 10.0.1.11
    db:
      hosts:
        db1:
          ansible_host: 10.0.2.20
```
### 1.2 Playbook Structure
#### `site.yaml`
```
- name: Install and configure web server
  hosts: web
  become: yes
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
```
### 1.3 Common Ansible Modules
#### `A. copy module`
```
- name: Copy index.html to web server
  copy:
    src: files/index.html
    dest: /var/www/html/index.html
    owner: www-data
    group: www-data
    mode: '0644'
```
#### `B. file module`
```
- name: Create a directory
  file:
    path: /opt/myapp
    state: directory
    mode: '0755'
```
#### `C. user module`
```
- name: Create application user
  user:
    name: appuser
    shell: /bin/bash
    groups: sudo
    state: present
```
#### `D. service module`
```
- name: Restart nginx
  service:
    name: nginx
    state: restarted
```
#### `E. package module (generic)`
```
- name: Install git
  package:
    name: git
    state: present
```
### 1.4 Ansible Roles (Standard Directory Structure)
```
myrole/
  tasks/
    main.yml
  handlers/
    main.yml
  templates/
    app.conf.j2
  files/
    index.html
  vars/
    main.yml
  defaults/
    main.yml
  meta/
    main.yml
```
#### `Sample: tasks/main.yml`
```
- name: Install nginx
  apt:
    name: nginx
    state: present

- name: Deploy config
  template:
    src: app.conf.j2
    dest: /etc/nginx/sites-enabled/app.conf
  notify: restart nginx
```
#### `Sample: handlers/main.yml`
```
- name: restart nginx
  service:
    name: nginx
    state: restarted
```
#### `Using the role in a playbook`
```
- name: Deploy web app
  hosts: web
  roles:
    - myrole
```
### 1.5 Variable Precedence (Most important topic)
#### `Order from LOWEST → HIGHEST priority`
#### 1. Role defaults (`roles/role/defaults/main.yml`)
#### 2. Inventory variables (group → host)
#### 3. Playbook variables
#### 4. Block vars
#### 5. Task vars
#### 6. Extra vars (`--extra-vars`) ← Highest priority
#### Example:
```
ansible-playbook site.yaml --extra-vars "version=2.0"
```
#### `version=2.0 overrides every other place where `version` is defined.
### 1.6 Templates (Jinja2)
#### `Nginx config template – app.conf.j2`
```
server {
    listen 80;
    server_name {{ app_domain }};

    root /var/www/html/{{ app_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
#### `Playbook using template`
```
- name: Apply nginx template
  template:
    src: app.conf.j2
    dest: /etc/nginx/sites-enabled/app.conf
  vars:
    app_domain: example.com
    app_name: myapp
```
