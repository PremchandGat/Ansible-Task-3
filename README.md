# Ansible-Task-3

# Create a Folder for roles
<pre>
[root@localhost ~]# cd /
[root@localhost ~]# mkdir myroles
[root@localhost /]# cd  myroles/
</pre>
# Create Role to launch 3 instance for httpd and 1 for HAProxy on AWS
<pre>
[root@localhost myroles]# ansible-galaxy init launch-instance
- Role launch-instance was created successfully
[root@localhost myroles]# cd launch-instance/
[root@localhost launch-instance]# ls
defaults  files  handlers  meta  README.md  tasks  templates  tests  vars
[root@localhost launch-instance]# cd tasks
[root@localhost tasks]# vim main.yml
</pre>
This is task main file of launch-instance Role
<pre>
---

# tasks file for launch-instance for
- name: Launching the Webservers EC2 instances on AWS
  ec2:
    region: "ap-south-1"
    image: "ami-052c08d70def0ac62"
    instance_type: "t2.micro"
    count: 3 
    vpc_subnet_id: "{{ subnet }}"
    group_id: "{{ sg }}"
    instance_tags:
      Name: "webserver"
    key_name: "{{ key }}"
    assign_public_ip: yes
    state: present
    wait: yes
    aws_access_key: "{{ access }}"
    aws_secret_key: "{{ secret }}"
  register: webserver_logs

- name: Launching the load balancer EC2 instances on AWS
  ec2:
    region: "ap-south-1"
    image: "ami-052c08d70def0ac62"
    instance_type: "t2.micro"
    count: 1 
    vpc_subnet_id: "{{ subnet }}"
    group_id: "{{ sg }}"
    instance_tags:
      Name: "ha"
    key_name: "{{ key }}"
    assign_public_ip: yes
    state: present
    wait: yes
    aws_access_key: "{{ access }}"
    aws_secret_key: "{{ secret }}"
  register: ha_logs


- debug:
     var: webserver_logs
     var: ha_logs

</pre>
<pre>
[root@localhost tasks]# cd ../vars/
[root@localhost vars]# vim main.yml 
</pre>
This is  vars main file of launch-instance Role
<pre>
---
# vars file for launch-instance
secret: "Write your secret key"
access: "Write your access key"
key: "mykey"
subnet: "subnet-7eccc816"                               
sg: "sg-078531d324434dcae"
</pre>
# create a Role for configuring Httpd webserver 
<pre>
[root@localhost /]# cd myroles/
[root@localhost myroles]# ansible-galaxy init conf-webserver
- Role conf-webserver was created successfully
[root@localhost myroles]# cd conf-webserver/
[root@localhost conf-webserver]# ls
defaults  files  handlers  meta  README.md  tasks  templates  tests  vars
[root@localhost conf-webserver]# cd tasks/
[root@localhost tasks]# vim main.yml
</pre>
This is tasks main file for conf-webserver Role
<pre>
---
# tasks file for conf-webserver                                                                       
- name: Installing Apache Web Server (httpd)
  package:
    name: httpd
    state: present

- name: Copying webpage
  copy:
    content: "Configured by Ansible  Premchand {{ ansible_hostname }}"                  
    dest: "/var/www/html/index.html"
    mode: 644

- name: Enabling and starting the web server
  service:
    name: "httpd"
    state: started
    enabled: yes
</pre>
# Configure HAProxy using Ansible Role
<pre>
[root@localhost /]# cd myroles/
[root@localhost myroles]# ansible-galaxy init conf-ha
- Role conf-ha was created successfully
[root@localhost myroles]# cd conf-ha/
[root@localhost conf-webserver]# ls
defaults  files  handlers  meta  README.md  tasks  templates  tests  vars
[root@localhost conf-webserver]# cd tasks/
[root@localhost tasks]# vim main.yml
</pre>
This is tasks main file for conf-ha Role
<pre>
---
# tasks file for conf-ha
- name: Installing load balancer HAProxy
  package:
     name: "haproxy"
     state: present

- name: Copying HAProxy's configuration file
  template:
    src: "haproxy.cfg"
    dest: "/etc/haproxy/haproxy.cfg"
    mode: preserve
  notify: restart HAProxy

- name: Starting the load balancer (HAProxy)
  service:
    name: "haproxy"
    state: started
</pre>
<pre>
[root@localhost tasks]# cd ../handlers/
[root@localhost handlers]# ls
main.yml
[root@localhost handlers]# vim main.yml
</pre>
This is handlers main file for conf-ha Role
<pre>
---
# handlers file for conf-ha
- name: restart HAProxy
  service:
     name: "haproxy"
     state: restarted
</pre>
<pre>
[root@localhost handlers]# cd ../templates/
[root@localhost templates]# ls
haproxy.cfg
[root@localhost templates]# vim haproxy.cfg
</pre>
This is template file for HAProxy 
<pre>
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/1.8/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend main
    bind *:80
    acl url_static       path_beg       -i /static /images /javascript /stylesheets
    acl url_static       path_end       -i .jpg .gif .png .css .js

    use_backend static          if url_static
    default_backend             app

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static
    balance     roundrobin
    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    server  app1 127.0.0.1:5001 check
    {% for i in groups ['tag_Name_webserver'] %}
    server  app1 {{i}}:80 check
    {% endfor %}


</pre>
# Create a inventory folder 
<pre>
[root@localhost tasks]# cd /
[root@localhost /]# mkdir ip
[root@localhost /]# cd ip
[root@localhost ip]# wget https://raw.githubusercontent.com/Premchandg278/Ansible-Task-2/main/ip/ec2.ini
[root@localhost ip]# wget https://raw.githubusercontent.com/Premchandg278/Ansible-Task-2/main/ip/ec2.py
[root@localhost ip]# chmod +x ec2.ini
[root@localhost ip]# chmod +x ec2.py
[root@localhost ip]# ls
ec2.ini  ec2.py
</pre>
# Create a ansible playbook for task-3
<pre>
[root@localhost /]# vim launch.yml
this playbook launch instance

- hosts: localhost
  roles:
          - role:  launch-instance 


[root@localhost /]# vim conf.yml
this playbook configure web server and HAProxy(load-balancer)

- hosts: tag_Name_webserver
  roles:
    - role: conf-webserver

- hosts: tag_Name_ha
  roles:
    - role: conf-ha
</pre>
# Configure Ansible 
<pre>
[root@localhost /]# vim /etc/ansible/ansible.cfg
</pre>
ansible.cfg file
<pre>
[defaults]
inventory=/ip/
roles_path=/myroles
host_key_checking = False
private_key_file = /mykey.pem
remote_user = ec2-user
ask_pass = False
become = TRUE

[privilege_escalation]
become = TRUE
become_user = root
become_method = sudo
become_ask_pass = FALSE
</pre>

# Download your ssh key of aws and give some permissions
<pre>
[root@localhost /]# chmod 400 mykey.pem
[root@localhost /]# chmod 600 mykey.pem
[root@localhost /]# ls -l
-rw-------.   1 root root 1674 Jul 30 23:37 mykey.pem
</pre>
# Declare AWS access key , secret key and region
<pre>
[root@localhost /]# export AWS_REGION=ap-south-1
[root@localhost /]# export AWS_ACCESS_KEY_ID=AKIAYXXXXXXXXXXXXXXX
[root@localhost /]# export AWS_SECRET_ACCESS_KEY=vDDg94XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
</pre>

# Finally run this command
<b>[root@localhost /]# ansible-playbook launch.yml</b>
<b>[root@localhost /]# ansible-playbook conf.yml</b>
