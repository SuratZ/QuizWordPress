---
- name: Install Wordpress by Ansible
  hosts: all
  gather_facts: false
  remote_user: root
  vars:
     ansible_password: redhat
  tasks:
  - name: loop install all package
    yum:
     name: "{{ item.name }}"
     state: latest
    loop:
     - {name: httpd}
     - {name: mariadb}
     - {name: mariadb-server}
     - {name: php}
     - {name: php-common}
     - {name: php-mysql}
     - {name: php-gd}
     - {name: php-xml}
     - {name: php-mbstring}
     - {name: php-mcrypt}
     - {name: php-xmlrpc}
     - {name: unzip}
     - {name: wget}

  - name: Start service httpd, if not started
    service:
     name: "{{ item.name }}"
     state: started
     enabled: true 
    loop:
     - {name: httpd}
     - {name: mariadb}

  - name: enable firewalld
    firewalld:
     service: http
     permanent: true
     state: enabled
     immediate: yes

  - name: Set SQL
    command: echo "{{ item }}" | mysql -uroot -p"abc"
    loop:
     - "DELETE FROM mysql.user WHERE User='';"
     - "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost','127.0.0.1', '::1');"
     - "DROP DATABASE test;"
     - "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%';"
     - "FLUSH PRIVILEGES;"
     - "CREATE DATABASE wordpress"
     - "GRANT ALL PRIVILEGES on wordpress.* to 'ansible'@'localhost' identified by 'mypassword';"
     - "FLUSH PRIVILEGES;"
    
  - name: fetch a file and unarchive
    unarchive:
     src: https://wordpress.org/wordpress-5.0.tar.gz
     dest: /var/www/html/
     remote_src: yes 

  - name: change ownership, permission
    file:
     path: /var/www/html/wordpress
     owner: apache
     group: apache
     mode: 755

  - name: create directory
    file:
     path: /var/www/html/wordpress/wp-content/uploads
     state: directory
     group: apache

  - name: Change name .php
    copy:
     remote_src: yes
     src: /var/www/html/wordpress/wp-config-sample.php
     dest: /var/www/html/wordpress/wp-config.php
     

  - name: Update WordPress config file
    lineinfile:
      dest=/var/www/html/wordpress/wp-config.php
      regexp="{{ item.regexp }}"
      line="{{ item.line }}"
    with_items:
      - {'regexp': "define\\('DB_NAME', '(.)+'\\);", 'line': "define('DB_NAME', 'wordpress');"}        
      - {'regexp': "define\\('DB_USER', '(.)+'\\);", 'line': "define('DB_USER', 'ansible');"}        
      - {'regexp': "define\\('DB_PASSWORD', '(.)+'\\);", 'line': "define('DB_PASSWORD', 'mypassword');"}
      - {'regexp': "define\\('DB_HOST', '(.)+'\\);", 'line': "define('DB_HOST', 'localhost');"}
    become: yes
