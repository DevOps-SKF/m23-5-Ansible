## 23.5.1 Ansible Playbook with tags and vault

## Задача 

1. Напишите плейбук, который будет на удаленной системе создавать пользователя user1 и задавать ему SSH-ключ (создавая заданный ключ в /home/user1/.ssh/id_rsa.pub). Ключ не должен передаваться в открытом виде. Используйте для шифрования ключа ansible-vault. Ключ можете сгенерировать, используя SSH-keygen.
2. Создайте плейбук, который устанавливает, либо удаляет почтовый сервер postfix в зависимости от тега.
3. Поиграть с тегами.

## Решение

### 1 - vault

Пароль от vault храню в файле, чтобы не вводить и вообще о нем не думать.

`/etc/ansible/ansible.cfg`:

    [defaults]
    ### !-! my variables
    vault_password_file=~/.ssh/ansible_vault

`ansible-vault encrypt ~/.ssh/usertest.pub`

Теперь содержимое файла:

    $ANSIBLE_VAULT;1.1;AES256
    35366165613562316335303435303934646231643434336137363262623034613135366461323766
    6563653566623436346365353533656464646637366666630a383162383437373632646161326136


`ansible-playbook -i hosts m23-5.yml --tags user`

На целевой машине файл расшифрован:

    ssh-rsa AAAAB3Nz


### 2 - postfix

Запуск:  
`ansible-playbook -i hosts m23-5.yml -t postfix_init`  
`ansible-playbook -i hosts m23-5.yml -t postfix_drop[,debug]`  

При установке `postfix` задается ряд вопросов. Использую модуль `debconf` для подготовки и подстановки ответов.

Результат:

    anton@skfubu:~$ telnet localhost 25
    Trying 127.0.0.1...
    Connected to localhost.
    Escape character is '^]'.
    220 skfubu.pvt.arlab.pw ESMTP Postfix (Ubuntu)


### 3 - tags

Я хочу, чтобы при наличии тега `user` выполнялся соответствующий блок.  
При наличии тегов `user,debug` должна выводиться дополнительная отладочная информация в этом блоке.

Но тэги по умолчанию работают как "любой".

    - name: User
        block: 
        - debug: msg="Creating User"

        - debug: msg="Creating User debug"
          tags: debug
        tags: user

`ansible-playbook -i hosts m23-5.yml --tags user`

    ok: [192.168.5.100] => {
        "msg": "Creating User"
    }

    ok: [192.168.5.100] => {
        "msg": "Creating User debug"
    }

Нижний не должен присутствовать.  
Поэтому заменяю тэг `debug` на условие

    - name: User
        block: 
        - debug: msg="Creating User"

        - debug: msg="Creating User debug"
          when:
            - '"debug" in ansible_run_tags'
        tags: user

`ansible-playbook -i hosts m23-5.yml --tags user`

    ok: [192.168.5.100] => {
        "msg": "Creating User"
    }

    TASK [debug] 
    skipping: [192.168.5.100]

`ansible-playbook -i hosts m23-5.yml --tags user,debug`

    ok: [192.168.5.100] => {
        "msg": "Creating User"
    }

    TASK [debug] 
    ok: [192.168.5.100] => {
        "msg": "Creating User debug"
    }

`ansible-playbook -i hosts m23-5.yml --tags debug`

    PLAY [all] 

    PLAY RECAP 

Теперь все, как задумано
