1. nano roles/server/tasks/main.yml

---
- name: set locale
  shell: export LC_ALL="en_US.UTF-8" && export LANGUAGE="en_US.UTF-8" && locale >> /etc/environment
  sudo: yes

- name: update apt cache
  apt: update_cache=yes cache_valid_time=3600
  sudo: yes

- name: install required software
  apt: pkg={{ item }} state=latest
  sudo: yes
  with_items:
    - curl
    - gnupg
    - fail2ban
    - ufw
    - wget
    - rsync
    - git
    - zip
    - build-essential
    - vim
    - chkrootkit
    - psmisc

- name: ensure fail2ban is running
  sudo: yes
  action: service name=fail2ban state=restarted enabled=yes

- name: forbid SSH root login
  sudo: yes
  lineinfile: destfile=/etc/ssh/sshd_config regexp="^PermitRootLogin" line="PermitRootLogin no" state=present
  notify: restart ssh

- name: reset firewall
  sudo: yes
  action: shell ufw --force reset

- name: allow firewall authorized ports
  sudo: yes
  action: shell ufw allow {{ item }}
  with_items:
  - 22
  - 80

- name: enable firewall
  sudo: yes
  action: shell ufw --force enable



2. nano roles/nginx/templates/default.conf

server {
        listen       80 default_server;
        root /var/www/wordpress;

        client_max_body_size 64M;

        # Deny access to any files with a .php extension in the uploads directory
        location ~* /(?:uploads|files)/.*\.php$ {
                deny all;
        }

        location / {
                index index.php index.html index.htm;
                try_files $uri $uri/ /index.php?$args;
        }

        location ~* \.(gif|jpg|jpeg|png|css|js)$ {
                expires max;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_index index.php;
                fastcgi_pass  unix:/var/run/php5-fpm.sock;
                fastcgi_param   SCRIPT_FILENAME
                                $document_root$fastcgi_script_name;
                include       fastcgi_params;
        }
}


3. nano roles/server/handlers/main.yml


---
- name: restart ssh
  sudo: yes
  action: service name=ssh state=restarted enabled=yes



nano roles/nginx/tasks/main.yml                                                       

---
- name: Update apt cache
  apt: update_cache=yes cache_valid_time=3600
  sudo: yes

- name: Install nginx
  apt: name={{ item }} state=present
  sudo: yes
  with_items:
    - nginx
  notify: restart nginx

- name: Remove default configuration nginx
  file: path=/etc/nginx/sites-enabled/default state=absent
  notify: restart nginx
  sudo: yes

- name: Copy nginx configuration for wordpress
  template: src=default.conf dest=/etc/nginx/conf.d/default.conf
  notify: restart nginx
  sudo: yes


4. nano roles/nginx/handlers/main.yml 


---
- name: restart nginx
  action: service name=nginx state=restarted enabled=yes
  sudo: yes


nano roles/php5-fpm/tasks/main.yml                                                    

---
- name: Update apt cache
  apt: update_cache=yes cache_valid_time=3600
  sudo: yes

- name: Install required software
  apt: name={{ item }} state=present
  sudo: yes
  with_items:
    - php5-mysql
    - php5
    - php5-fpm
    - php5-xmlrpc
    - php5-mcrypt
    - python-mysqldb

- name: Set maximum process manager children
  sudo: yes
  lineinfile: destfile=/etc/php5/fpm/pool.d/www.conf regexp="^pm.max_children" line="pm.max_children = 10" state=present

- name: Set number children requests
  sudo: yes
  replace: destfile=/etc/php5/fpm/pool.d/www.conf regexp="^;pm.max_requests = 500" replace="pm.max_requests = 500"
  notify: restart php5-fpm


5. nano roles/php5-fpm/handlers/main.yml                                                 

---
- name: restart php5-fpm
  action: service name=php5-fpm state=restarted enabled=yes
  sudo: yes


6. nano roles/mysql/tasks/main.yml 

---
- name: Update apt cache
  apt: update_cache=yes cache_valid_time=3600
  sudo: yes

- name: Install mysql server
  apt: name={{ item }} state=present
  sudo: yes
  with_items:
    - mysql-server

- name: Create mysql database
  mysql_db: name={{ wp_mysql_db }} state=present
  sudo: yes

- name: Create mysql user
  mysql_user:
    name={{ wp_mysql_user }}
    password={{ wp_mysql_password }}
    priv=*.*:ALL
  sudo: yes


7. nano roles/mysql/defaults/main.yml                                                    

---
wp_mysql_db: wordpress
wp_mysql_user: wordpress
wp_mysql_password: syalala


8. nano roles/wordpress/tasks/main.yml                                                   

---
- name: Download WordPress
  get_url:
    url=https://wordpress.org/latest.tar.gz
    dest=/tmp/wordpress.tar.gz
    validate_certs=no
  sudo: yes

- name: Extract WordPress
  unarchive: src=/tmp/wordpress.tar.gz dest=/var/www/ copy=no owner=www-data group=www-data
  sudo: yes

- name: Copy sample config file
  command: mv /var/www/wordpress/wp-config-sample.php /var/www/wordpress/wp-config.php creates=/var/www/wordpress/wp-config.php
  sudo: yes

- name: Update WordPress config file
  lineinfile:
    dest=/var/www/wordpress/wp-config.php
    regexp="{{ item.regexp }}"
    line="{{ item.line }}"
  with_items:
    - {'regexp': "define\\('DB_NAME', '(.)+'\\);", 'line': "define('DB_NAME', '{{wp_mysql_db}}');"}
    - {'regexp': "define\\('DB_USER', '(.)+'\\);", 'line': "define('DB_USER', '{{wp_mysql_user}}');"}
    - {'regexp': "define\\('DB_PASSWORD', '(.)+'\\);", 'line': "define('DB_PASSWORD', '{{wp_mysql_password}}');"}
  sudo: yes

