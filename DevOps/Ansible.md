[TOC]



# Playbook

## 关键字段

**playbook使用yaml来编排一套play，每个play调用ansible标准模块或自定义模块。这样组合起来，整体实现一定的功能。**

### Hosts

playbook中的每一个play的目的都是为了让特定主机以某个指定的身份执行任务。hosts用于指定主机，主机需要事先定义在主机清单中。

```yaml
- hosts: a:b //a集或b集。如果是与，则使用”a:&b“；如果是在a但不在b，则使用”a:!b“
```

### Tasks

play的主体部分是task list，task按次序逐个在hosts上执行。每个task应该有name，用于playbook的执行结果输出。

```yaml
- hosts: a
  remote_user: root
  gather_facts: no // 关闭收集主机信息，跳过set_up这一步，能加速。
  tasks:
    - name: install httpd
      yum: name=httpd state=present // module: args
    - name: copy configure file
      copy: src=./httpd.conf dest=/etc/httpd/conf  
    - name: start httpd
      service: name=httpd state=started enabled=yes
```

### Handlers，Notify

Notify中列出的操作称为Handler，Handler的语法与task一致。

Notify对应的Handler可用于在每个play的最后被触发，这样可以避免多次有改变发生时每次都执行指定的操作，仅在所有的变化发生完成后一次性执行指定操作。

比如：更改配置文件，重新执行ansible-playbook example.yml，则重新执行opy configure file并触发restart httpd。实现更新配置文件并重启服务的目的。

```yaml
- hosts: a
  remote_user: root
  gather_facts: no 
  tasks:
    - name: install httpd
      yum: name=httpd state=present 
    - name: copy configure file
      copy: src=./httpd.conf dest=/etc/httpd/conf 
      notify: restart httpd
    - name: start httpd
      service: name=httpd state=started enabled=yes
      
  handlers:
    - name: restart httpd
      service: name=httpd state=restarted
```

### Tags

给特定的task指定tag，执行playbook时，可以只执行特定tag的task，而不是整个playbook。

```yml
- hosts: a
  remote_user: root
  gather_facts: no 
  tasks:
    - name: install httpd
      yum: name=httpd state=present 
    - name: copy configure file
      copy: src=./httpd.conf dest=/etc/httpd/conf 
      tags: conf
    - name: start httpd
      service: name=httpd state=started enabled=yes
      tags: service
      
  # ansible-playbook -t conf,service example.yml    
```

### Variables

变量名仅能由字母，数字和下划线组成，只能以字母开头。

- 变量定义：

  key=value，比如：http_port=80。

- 变量引用：

  {{ key }}，有时使用"{{ key }}"才生效。

- 变量来源：

  1. ansible的setup facts远程主机的所有变量都可直接引用。setup模块可以获取主机的系统信息。

     ```yaml
     - hosts: all
       remote_user: root
       gather_facts: yes
       tasks:
         - name: create log file
           file: name=/data/{{ ansible_nodename }}.log state=touch owner=wang mode=600
     ```

     

  2. 通过命令行指定变量，优先级最高。

     ```yaml
     - hosts: all
       remote_user: root
       tasks: 
         - name: install package
           yum: name={{ pkgname }} state=present
     # ansible-playbook -e pkgname=memcached example.yml
     ```

  3. 在playbook文件中定义，优先级最低。

     ```yaml
     - hosts: all
       remote_user: root
       vars:
         - var1: val1
         - var2: val2
     ```

  4. 在独立的变量yaml文件中定义，优先级比playbook中的高。

     ```yaml
     vim vars.yml
     package_name: vsftpd
     service_name: vsftpd
     
     vim install_app.yml
     - hosts: a
       vars_files:
         - vars.yml
       tasks:
         - name: install
           yum: name={{ package_name }}
         - name: start
           service: name={{ service_name }} state=started enabled=yes
          
     ```

### Templates

目录结构：template文件必须位于templates目录下，且以.j2结尾。yaml/yml文件必须与templates目录平级。

```shell
./
|----temnginx.yml
|----templates
     |----nginx.conf.j2
```

举例：

- 根据不同主机的cpu核数，动态的配置nginx。

  ```yaml
  vim templates/nginx.conf.j2
  ...;
  worker_processes {{ ansible_processor_vcpus }};
  ...;
  
  vim temnginx.yml
  ---
  - hosts: a
    remote_user: root
    tasks:
      - name: install nginx
        yum: name=nginx
      - name: template config to remote hosts
        template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
      - name: start service
        service: name=nginx state=started enabled=yes
        
  ansible-playbook temnginx.yml --limit x.x.x.x      
  ```

### When

作为task执行的前提条件。

举例：

- 根据操作系统版本，判断是否重启nginx。

  ```yaml
  ---
  - hosts: a
    remote_user: root
    tasks:
      - name: install nginx
        yum: name=nginx state=present
      - name: restart service
        service: name=nginx state=restarted
        when: ansible_distribution_major_version == "6"
  ```

### With_Items

在playbook中，进行循环迭代，生成类似的task。

举例：

- 添加多个user。

  ```yaml
  - hosts: a
    tasks:
      - name: add several users
        user: name={{ item }} state=present groups=wheel
        with_items:
          - user1
          - user2
          
  # 上面语句等同于:
      - name: add several users
        user: name=user1 state=present groups=wheel
      - name: add several users
        user: name=user2 state=present groups=wheel
  ```

  



# Role