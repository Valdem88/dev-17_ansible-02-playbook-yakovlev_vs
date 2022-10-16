# Домашнее задание к занятию "Работа с Playbook" dev-17_ansible-02-playbook-yakovlev_vs
Работа с Playbook

## Подготовка к выполнению

1. (Необязательно) Изучите, что такое [clickhouse](https://www.youtube.com/watch?v=fjTNS2zkeBs) и [vector](https://www.youtube.com/watch?v=CgEhyffisLY)
2. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
3. Скачайте [playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.
4. Подготовьте хосты в соответствии с группами из предподготовленного playbook.

## Основная часть

1. Приготовьте свой собственный inventory файл `prod.yml`.
2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает [vector](https://vector.dev).
3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.
4. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, установить vector.
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
9. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
10. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-02-playbook` на фиксирующий коммит, в ответ предоставьте ссылку на него.

#### Решение

1.
```yaml
---
clickhouse:
  hosts:
    clickhouse-01:
      ansible_host: 192.168.1.221
      ansible_user: root

vector:
  hosts:
    vector-01:
      ansible_host: 192.168.1.222
      ansible_user: root
```
2.
```yaml
---
- name: Install Clickhouse
  hosts: clickhouse
  handlers:
    - name: Start clickhouse service
      become: true
      ansible.builtin.service:
        name: clickhouse-server
        state: restarted
  tasks:
    - block:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/{{ item }}-{{ clickhouse_version }}.noarch.rpm"
            dest: "./{{ item }}-{{ clickhouse_version }}.rpm"
          with_items: "{{ clickhouse_packages }}"
      rescue:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-{{ clickhouse_version }}.x86_64.rpm"
            dest: "./clickhouse-common-static-{{ clickhouse_version }}.rpm"
    - name: Install clickhouse packages
      become: true
      ansible.builtin.yum:
        name:
          - clickhouse-common-static-{{ clickhouse_version }}.rpm
          - clickhouse-client-{{ clickhouse_version }}.rpm
          - clickhouse-server-{{ clickhouse_version }}.rpm
      notify: Start clickhouse service
    - name: Flush handlers
      meta: flush_handlers
    - name: Create database
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc !=82
      changed_when: create_db.rc == 0

- name: Install Vector
  hosts: vector
  handlers:
  - name: Start Vector service
    become: true
    ansible.builtin.service:
      name: vector
      state: started

  tasks:
    - name: Get Vector distrib
      ansible.builtin.get_url:
        url: "https://packages.timber.io/vector/0.21.1/vector-0.21.1-1.{{ ansible_architecture }}.rpm"
        dest: "./vector-0.21.1-1.{{ ansible_architecture }}.rpm"

    - name: Install Vector packages
      become: true
      ansible.builtin.yum:
        name: vector-0.21.1-1.{{ ansible_architecture }}.rpm
      notify: Start Vector service
```
3.
Использовались `get_url`, `template`

4. playbook прописан так, чтобы значение версии для vector бралось из файла переменных - `vars.yml` 

5. Исправил ошибки. ansible-lint site.yml прошел.
```bash
root@ansibleserv:~/ansible/playbook# ansible-lint site.yml
root@ansibleserv:~/ansible/playbook#
```
6.
При запуске с ключем --check ansible ругается на отсутствие "скаченных" файлов.

7.
```bash
root@ansibleserv:~/ansible/playbook# ansible-playbook -i inventory/prod.yml site.yml --diff --ask-pass
SSH password:

PLAY [Install Clickhouse] *********************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************
ok: [clickhouse-01]

TASK [Get clickhouse distrib] *****************************************************************************************************************
changed: [clickhouse-01] => (item=clickhouse-client)
changed: [clickhouse-01] => (item=clickhouse-server)
failed: [clickhouse-01] (item=clickhouse-common-static) => {"ansible_loop_var": "item", "changed": false, "dest": "./clickhouse-common-static-2  2.3.3.44.rpm", "elapsed": 0, "item": "clickhouse-common-static", "msg": "Request failed", "response": "HTTP Error 404: Not Found", "status_code  ": 404, "url": "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-22.3.3.44.noarch.rpm"}

TASK [Get clickhouse distrib] *****************************************************************************************************************
changed: [clickhouse-01]

TASK [Install clickhouse packages] ************************************************************************************************************
changed: [clickhouse-01]

TASK [Flush handlers] *************************************************************************************************************************

RUNNING HANDLER [Start clickhouse service] ****************************************************************************************************
changed: [clickhouse-01]

TASK [Create database] ************************************************************************************************************************
changed: [clickhouse-01]

PLAY [Install Vector] *************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************
ok: [vector-01]

TASK [Get Vector distrib] *********************************************************************************************************************
changed: [vector-01]

TASK [Install Vector packages] ****************************************************************************************************************
changed: [vector-01]

RUNNING HANDLER [Start Vector service] ********************************************************************************************************
ok: [vector-01]

PLAY RECAP ************************************************************************************************************************************
clickhouse-01              : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=1    ignored=0
vector-01                  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
```bash
[root@clickhouse-01 ~]# clickhouse-client
ClickHouse client version 22.3.3.44 (official build).
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 22.3.3 revision 54455.

clickhouse-01 :) show databases

SHOW DATABASES

Query id: 9b9b66c3-469b-402b-8a65-a382124054a7

┌─name───────────────┐
│ INFORMATION_SCHEMA │
│ default            │
│ information_schema │
│ logs               │
│ system             │
└────────────────────┘

5 rows in set. Elapsed: 0.065 sec.
```
```bash
[root@vector-01 ~]# systemctl status vector
● vector.service - Vector
   Loaded: loaded (/usr/lib/systemd/system/vector.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-10-16 11:45:39 UTC; 13s ago
     Docs: https://vector.dev
  Process: 2247 ExecStartPre=/usr/bin/vector validate (code=exited, status=0/SUCCESS)
 Main PID: 2251 (vector)
   CGroup: /system.slice/vector.service
           └─2251 /usr/bin/vector

Oct 16 11:45:43 vector-01 vector[2251]: {"appname":"ahmadajmi","facility":"local5","hostname":"for.com","message":"#hugops to everyo...sion":2}
Oct 16 11:45:44 vector-01 vector[2251]: {"appname":"meln1ks","facility":"auth","hostname":"names.net","message":"Maybe we just shoul...sion":1}
Oct 16 11:45:45 vector-01 vector[2251]: {"appname":"jesseddy","facility":"daemon","hostname":"up.org","message":"There's a breach in...sion":1}
Oct 16 11:45:46 vector-01 vector[2251]: {"appname":"shaneIxD","facility":"local0","hostname":"random.org","message":"You're not gonn...sion":1}
Oct 16 11:45:47 vector-01 vector[2251]: {"appname":"jesseddy","facility":"clockd","hostname":"up.com","message":"Great Scott! We're ...sion":1}
Oct 16 11:45:48 vector-01 vector[2251]: {"appname":"shaneIxD","facility":"local5","hostname":"random.net","message":"Take a breath, ...sion":1}
Oct 16 11:45:49 vector-01 vector[2251]: {"appname":"devankoshal","facility":"local2","hostname":"names.org","message":"Pretty pretty...sion":2}
Oct 16 11:45:50 vector-01 vector[2251]: {"appname":"Karimmove","facility":"lpr","hostname":"for.us","message":"You're not gonna beli...sion":1}
Oct 16 11:45:51 vector-01 vector[2251]: {"appname":"Karimmove","facility":"audit","hostname":"some.com","message":"Maybe we just sho...sion":1}
Oct 16 11:45:52 vector-01 vector[2251]: {"appname":"meln1ks","facility":"ftp","hostname":"for.org","message":"A bug was encountered ...sion":2}
Hint: Some lines were ellipsized, use -l to show in full.
```
8. 
```bash
root@ansibleserv:~/ansible/playbook# ansible-playbook -i inventory/prod.yml site.yml --diff --ask-pass
SSH password:

PLAY [Install Clickhouse] ***********************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************
ok: [clickhouse-01]

TASK [Get clickhouse distrib] *******************************************************************************************************************
ok: [clickhouse-01] => (item=clickhouse-client)
ok: [clickhouse-01] => (item=clickhouse-server)
failed: [clickhouse-01] (item=clickhouse-common-static) => {"ansible_loop_var": "item", "changed": false, "dest": "./clickhouse-common-static-22.3.3.44.rpm", "elapsed": 0, "gid": 0, "group": "root", "item": "clickhouse-common-static", "mode": "0644", "msg": "Request failed", "owner": "root", "response": "HTTP Error 404: Not Found", "secontext": "unconfined_u:object_r:admin_home_t:s0", "size": 246310036, "state": "file", "status_code": 404, "uid": 0, "url": "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-22.3.3.44.noarch.rpm"}

TASK [Get clickhouse distrib] *******************************************************************************************************************
ok: [clickhouse-01]

TASK [Install clickhouse packages] **************************************************************************************************************
ok: [clickhouse-01]

TASK [Flush handlers] ***************************************************************************************************************************

TASK [Create database] **************************************************************************************************************************
ok: [clickhouse-01]

PLAY [Install Vector] ***************************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************
ok: [vector-01]

TASK [Get Vector distrib] ***********************************************************************************************************************
ok: [vector-01]

TASK [Install Vector packages] ******************************************************************************************************************
ok: [vector-01]

PLAY RECAP **************************************************************************************************************************************
clickhouse-01              : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=1    ignored=0
vector-01                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

9. 
https://github.com/Valdem88/dev-17_ansible-02-playbook-yakovlev_vs/blob/main/step/readme.md

10.
https://github.com/Valdem88/dev-17_ansible-02-playbook-yakovlev_vs/blob/main/playbook/site.yml

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---