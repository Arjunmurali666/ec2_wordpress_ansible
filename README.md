# ec2_wordpress_ansible
## Ansible - This ansible playbook is written to launch an ec2 instance from AWS with WordPress installed on it. 

```
---
 - name: "EC2_LAMP"
   hosts: localhost
   gather_facts: False
   vars:
     domain: www.example.com
     mysql_root: mysqlroot123
     mysql_database: wodpress
     mysql_user: wpuser
     mysql_password: wpuser
        
   tasks:
     - name: "Launch new EC2"
       local_action: ec2
                     group=default
                     instance_type=t2.micro
                     image=ami-0d8f6eb4f641ef691
                     wait=true
                     region=us-east-2
                     keypair=ansible
                     count=1
       register: ec2

     - debug:
         var: item.public_ip
       with_items: "{{ ec2.instances }}"

     - name: "Adding host"
       add_host:
         hostname: webserver
         ansible_host: "{{ ec2.instances.0.public_ip }}"
         ansible_port: 22
         ansible_user: "ec2-user"
         ansible_ssh_private_key_file: "/home/ec2-user/ansible.pem"
         ansible_ssh_common_args: "-o StrictHostKeyChecking=no"

     - name: "wait for ec2 instance to come online"
       wait_for:
         port: 22
         host: "{{ ec2.instances.0.public_ip }}"
         timeout: 80
         state: started
         delay: 10
         
     - name: "Setup Apache webserver"
       delegate_to: webserver
       become: yes
       yum:
         name:
            - php-mysql
            - httpd
            - mariadb-server
            - php
            - MySQL-python

     - name: "Apache - Creating index.html"
       delegate_to: webserver
       become: yes
       copy:
         content: " <h1>It works..!!</h1>"
         dest: /var/www/html/index.html

     - name: "Apache - Restarting/Enabling"
       delegate_to: webserver
       become: yes
       service:
         name: "{{ item }}"
         state: restarted
       with_items:
         - httpd
         - mariadb
        
     - name: 'creating virtual host'
       delegate_to: webserver
       become: yes
       template:
         src: virtualhost.j2
         dest: /etc/httpd/conf.d/{{domain}}.conf
            
     - name: 'Mariadb-server Reset Root Password'
       delegate_to: webserver
       become: yes
       ignore_errors: true
       mysql_user:
         login_user: root
         login_password: ''
         name: root
         password: "{{mysql_root}}"
         host_all: true

     - name: 'Removing Anonymous users'
       delegate_to: webserver
       become: yes
       mysql_user:
         login_user: root
         login_password: "{{mysql_root}}"
         name: ''
         state: absent
         host_all: true

     - name: 'Creating database'
       delegate_to: webserver
       become: yes
       mysql_db:
         login_user: root
         login_password: "{{mysql_root}}"
         name: "{{mysql_database}}"
         state: present

     - name: 'Mariadb user add'
       delegate_to: webserver
       become: yes
       mysql_user:
         login_user: root
         login_password: "{{mysql_root}}"
         name: "{{mysql_user}}"
         password: "{{mysql_password}}"
         state: present
         host: localhost
         priv: "{{mysql_database}}.*:ALL"
            
     - name: 'creating document root'
       delegate_to: webserver
       become: yes
       file:
         path: /var/www/html/{{domain}}
         state: directory
         owner: apache
         group: apache   
                     
     - name: 'Downloading WordPress to /tmp'
       delegate_to: webserver
       become: yes
       get_url:
         url: https://wordpress.org/wordpress-4.9.2.tar.gz
         dest: /tmp/wordpress.tar.gz
            
     - name: 'Extracting contents from wordpress'
       delegate_to: webserver
       become: yes
       unarchive:
         src: /tmp/wordpress.tar.gz
         dest: /tmp/
         remote_src: yes
            
     - name: 'Copy wordpress content to document root'
       delegate_to: webserver
       become: yes
       shell: "cp -r /tmp/wordpress/*  /var/www/html/{{domain}}"
        
     - name: 'Wordpress - Creating wp-config.php'
       delegate_to: webserver
       become: yes
       template:
         src: wp-config.php.tmpl
         dest: /var/www/html/{{ domain }}/wp-config.php
          
     - name: 'Wordpress - Chaning Ownership'
       delegate_to: webserver
       become: yes
       shell: 'chown -R apache:apache /var/www/html/{{domain}}'
        
     - name: 'Deleting wordpress from /tmp'
       delegate_to: webserver
       become: yes
       file:
         path: "{{ item }}"
         state: absent
       with_items:
         - /tmp/wordpress.tar.gz
         - /tmp/wordpress/
        
     - name: "LampStack - Restarting Services"
       delegate_to: webserver
       become: yes
       service:
         name: "{{ item }}"
         state: restarted
       with_items:
         - httpd
         - mariadb
```

### Please save the following files as provided and run the ansible playbook

### virtualhost.j2

```
<virtualhost *:80>
  servername {{ domain }}
  documentroot /var/www/html/{{domain}}
  directoryindex index.php index.html info.html info.php
</virtualhost>
```

### wp-config.php.tmpl

```
<?php

define( 'DB_NAME', '{{ mysql_database }}' );

define( 'DB_USER', '{{ mysql_user }}' );

define( 'DB_PASSWORD', '{{ mysql_password }}' );

define( 'DB_HOST', 'localhost' );

define( 'DB_CHARSET', 'utf8' );

define( 'DB_COLLATE', '' );


define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );


$table_prefix = 'wp_';

define( 'WP_DEBUG', false );

if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', dirname( __FILE__ ) . '/' );
}

require_once( ABSPATH . 'wp-settings.php' );
```
### Now run the ansible playbook as follows

```
[ec2-user@ip-172-31-**-**~]$ ansible-playbook ec2-with-wp.yml 
 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match
'all'


PLAY [EC2_LAMP] **********************************************************************************************************

TASK [Launch new EC2] ****************************************************************************************************
changed: [localhost -> localhost]

TASK [debug] *************************************************************************************************************
ok: [localhost] => (item=None) => {
    "item.public_ip": "3.19.232.91"
}

TASK [Adding host] *******************************************************************************************************
changed: [localhost]

TASK [wait for ec2 instance to come online] ******************************************************************************
ok: [localhost]

TASK [Setup Apache webserver] ********************************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Apache - Creating index.html] **************************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Apache - Restarting/Enabling] **************************************************************************************
changed: [localhost -> 3.19.232.91] => (item=httpd)
changed: [localhost -> 3.19.232.91] => (item=mariadb)

TASK [creating virtual host] *********************************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Mariadb-server Reset Root Password] ********************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Removing Anonymous users] ******************************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Creating database] *************************************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Mariadb user add] **************************************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [creating document root] ********************************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Downloading WordPress to /tmp] *************************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Extracting contents from wordpress] ********************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Copy wordpress content to document root] ***************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Wordpress - Creating wp-config.php] ********************************************************************************
changed: [localhost -> 3.19.232.91]

TASK [Wordpress - Chaning Ownership] *************************************************************************************
 [WARNING]: Consider using file module with owner rather than running chown

changed: [localhost -> 3.19.232.91]

TASK [Deleting wordpress from /tmp] **************************************************************************************
changed: [localhost -> 3.19.232.91] => (item=/tmp/wordpress.tar.gz)
changed: [localhost -> 3.19.232.91] => (item=/tmp/wordpress/)

TASK [LampStack - Restarting Services] ***********************************************************************************
changed: [localhost -> 3.19.232.91] => (item=httpd)
changed: [localhost -> 3.19.232.91] => (item=mariadb)

PLAY RECAP ***************************************************************************************************************
localhost                  : ok=20   changed=18   unreachable=0    failed=0   
```

### Now you can access the domain in the browser using /etc/hosts file to get the WordPress install screen.
