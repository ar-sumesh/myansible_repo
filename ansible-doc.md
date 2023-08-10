# Ansible [learn linux tv]
## prep-configuration
- Make sure openssh in installed on the workstation and servers
- connect to each server form the workstation , answer "yes" to initial connection prompt.
- create an ssh key pair with passphrase for extra security.
- copy that key to each server.
- create an ssh key that is specific to ansible.
- copy that key to each server.

## server-details

```
server1 192.168.1.12 [workstation]
server2 192.168.1.8
server3 192.168.1.13
```

## create ssh keys
- stores under ~/.ssh
- generate key using ssh-keygen
- `-t` for type , `-C` for comment
```
ssh-keygen -t ed25519 -C "sumesh default"
```
- i given the passphrase :- `sumesh-sshkey`
- copy the ssh keys to both servers [server2, server3] from server1.
```
ssh-copy-id -i ~/.ssh/id_ed25519.pub 192.168.1.8
```

### create dedicated ansible ssh keys
```
ssh-keygen -t ed25519 -C "ansible"

```
- give the different path to save
```
/home/osboxes/.ssh/ansible
```
- Don't give the passphrase for the ssh key.
- now copy the keys
```
ssh-copy-id -i ~/.ssh/ansible.pub 192.168.1.8
```
- it will ask the password of the sshkey named [sumesh default] as the credentials.
- connecting to the ssh server by specifiing the ssh private key
```
 ssh -i ~/.ssh/ansible 192.168.1.8
```

## using ssh agent to store the ssh key
- create ssh-agent instance
```
eval $(ssh-agent)
```
- once you see the pid of the instance then add the passphrase
```
ssh-add
```
- then provide the ssh key

- NOTE:- this agent will exit if you close the terminal, so for every terminal session you have to create new ssh-agent instance.

### create alias for ssh-agent
- open `.bashrc`
- add the following line in any location
- `alias ssha='eval $(ssh-agent) && ssh-add'`
- so now in the new terminal you enter the command `ssha` this create new ssh-agent and ask for the password of the ssh key.


## Create git repo for ansible code

- git should be installed if not there (workstation).
- now add the ssh key in your github account, go to profile -> settings -> ssh and GPG keys
- click on new ssh key.
- copy and paste the public ssh key pair you created for the sumesh default.\
- clone the repo using ssh
- config the your details
```
git config --global user.name "Sumesh R Acharya"
git config --global user.email "sumesh@xerussystems.com"
```

## Installing ansible
- udpate the indexes and install ansible
```
sudo apt update
sudo apt install ansible
```
- install it in workstation system.

## ad-HOC commands
- create inventory file
  - which contains the ip address/information of the hosts.

- go under the `myansible_repo`
- nano `inventory`
```
192.168.1.8
192.168.1.13
```

- run the following command to perform initial ping
  - parameter info
  ```
  all -> against all hosts
  --key-file -> ssh private key to use
  -i -> path to the inventory file
  -m -> module to use
  ```
```
ansible all --key-file ~/.ssh/ansible -i inventory -m ping
```
- output should be
```
192.168.1.8 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
192.168.1.13 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

- create `ansible.cfg` ansible configuration file
- contents of the file
```
[defaults]
inventory = inventory
private_key_file = ~/.ssh/ansible
```
- now running the ansible command in the location where ansible.cfg file present.
- `ansible all -m ping`
- it should return the same output of like before

- gather_facts
```
ansible all -m gather_facts 

#limit to specific host

ansible all -m gather_facts --limit 192.168.1.8
```

## evalvated ad-hoc commmands
- Here we are trying to run the `apt update` command in the servers using ansible which sudo privileges.
```
ansible all -m apt -a update_cache=true --become --ask-become-pass
```
- `become` it tells to become super-privileges , `--ask-become-pass` will ask the super-user password.
- `-a` parameters for the module.
- this module will update the package indexes in the servers.
- NOTE:- sudo password in all the servers should be same.

### install packages
```
ansible all -m apt -a name=vim-nox --become --ask-become-pass
```
- name of the package we are installing is `vim-nox`
- To upgrade the package to latest version in the servers.
```
ansible all -m apt -a "name=curl state=latest" --become --ask-become-pass
```
- here i added the second argument where i define the state of the package to latest, the ansible will make that package version is latest.
- To upgrade all the packages in the servers.
```
ansible all -m apt -a "upgrade=dist" --become --ask-become-pass
```

## writing a ansible playbook
- command to run the playbook
```
 ansible-playbook --ask-become-pass install_apache.yml 
```
- in this playbook we are installing the apache2 in all the servers.



## using when conditional
- Added new server `Centos8`
```
server1 192.168.1.12 [workstation]
server2 192.168.1.8
server3 192.168.1.13
server4-centos 192.168.1.5
```
- update the mirror list , run the below commands with sudo
```
# sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
# sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```
- when condition is used to differentiate the os distribution when installing the packages in different os like ubuntu, centos etc.
- like in the above playbook i added a condition to run the install package command when the distro is ubuntu
```
when: ansible_distribution == "Ubuntu"
```
- when you have to add check condition for mixed distro:
```
when: ansible_distribution in ["Debian", "Ubuntu"]
```
- you can find these variable in the output of the `gather_facts`
- gather distribution information from the new system.
```
 ansible all -m gather_facts --limit 192.168.1.5 | grep "distribution"
```
- output look like
```
"ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "8",
        "ansible_distribution_release": "NA",
        "ansible_distribution_version": "8.3",
```

- add when condition for the centos
```
 when: ansible_distribution == "CentOS"
```
- also changed the package manager names and  package which are available for centos
- NOTE :- some manual commands has to be added to ran in the centos system as web server is not started running by default
```
#  systemctl start httpd
# firewall-cmd --add-port=80/tcp
```
- how to run these manual commands using ansible will be covered in the later section.

## improving your playbook 08

- first combine the installation of the package and updation of the package index in one plays
```
apt:
  name: 
    - apache2
    - libapache2-mod-php
  state: latest
  update_cache: yes
```
- Now removing the when condition from the plays and rename apt to `package` which is the inbuilt module of the ansible which uses the package manager based on the distro.
- Also create variable and combine the plays to single play.
- But you have to pass the value of the variable in the inventory corresponding to the server ip address
  ```
   # Install apache package
  - name: install apache2 package
    package:
      name: 
        - "{{ apache_package }}"
        - "{{ php_package }}"
      state: latest
      update_cache: yes
  ```
- inventory files changes
```
192.168.1.8 apache_package=apache2 php_package=libapache2-mod-php
192.168.1.13 apache_package=apache2 php_package=libapache2-mod-php
192.168.1.5  apache_package=httpd php_package=php
```

## Targeting nodes

- `pre_tasks` will be ran first then other tasks in the playbook.
- for targeting nodes i have added groups in the inventory file and in the playbook file we are calling the group name as the `hosts` , so that tasks are ran in those servers.


## Tags

- To list the tags in the playbook
```
ansible-playbook --list-tags site.yml
```
- To run the playbook using tags
```
ansible-playbook --tags centos --ask-become-pass site.yml
```
- Targeting multiple tags
```
ansible-playbook --tags "centos,database" --ask-become-pass site.yml
```

## Managing files

- I this section we are going to copy the files to the servers from the workstation to host servers in particular location.
- bydefault it searches under the `files` directory in workstation.
- we are using the `copy` module.
  - where we have to give following parameters.
  - src: name of the file kept under the files directory where the playbook is present.
  - dest: path of the server where you went to keep this files.
  - owner: owner of the root in the destination location `root`.
  - group: in which group this file belongs to.
  - mode: file permission.

- using `unarchive` module
  - src: url of the zip from the internet in this case it is `terraform`.
  - dest: /usr/local/bin
  - remote_src: yes [ we are telling ansible to not to look in the files in the current location in the system ]
  - mode: 0755
  - owner: root
  - group: root
  

## Managing services

- we will use the `service` module in ansible used for starting, stopping any service in the server.
```
service:
  name: httpd
  state: started
  enabled:yes
```
 - name :- service name
   state: started/stopped/restarted [ to start or stop the service]
   enabled: yes [ service should be enabled or not]

### restarting the service

- when any configuration is changed the service has to be restarted so that the new changes can be reflected.
- in below example we are going to make a line change in the httpd configuration then restarting the httpd service.
  ```
  # changing the httpd configuration file
  - name: change email address for admin
    tags: apache,centos,httpd
    lineinfile:
      path: /etc/httpd/conf/httpd.conf
      regexp: '^ServerAdmin' [finding the line starting with the text ]
      line: ServerAdmin sumesh@example.com [ content for that line]
    when: ansible_distribution == "CentOS"
    register: httpd # creating a new httpd variable

  # restarting the httpd service
  - name: restart the httpd (centos)
    tags: apache,centos,httpd
    service:
      name: httpd
      state: restarted
    when: httpd.changed  [checking the value of the variable created above]
  ```

- note if you using the same variable in the multiple plays then the value of the variable changes in each plays so it will not same which is stored when used first time.


## Adding users [13]

- here we will use the `user` module to create user in the server.
```
- hosts: all
  become: true
  tasks:
  
  - name: create simone user
    tags: always
    user:
      name: simone
      groups: root

  # adding authorized key and sudoer file for simone user
  - name: add ssh authorized key
    tags: always
    authorized_key:
      user: simone
      key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILka1m9fw9qKroEDSEtjwXuLxfzlbxdVwXWmXL3x+meT ansible"

  - name: add sudoers file for simone user
    tags: always
    copy:
      src: sudoer_simone
      dest: /etc/sudoers.d/simone
      owner: root
      group: root
      mode: 0440
```

- now adding the `authorized_key` and sudo permission for the simone user so that we can use the ssh to access the server using private key.

### removing the ask become pass
- we will use the above simone user to execute the superuser commands in the servers
- in the **ansible.cfg** we will mention which remote user we want to use
  ```
  remote_user = simone
  ```

### creating bootstrap playbook

- In this step following tasks will be taken place in the server:
  - run the update and install upgrade
  - create simone user
  - add the authorized key for the simone in all servers
  - add the sudoers file for simone user in the servers.
- for running this bootstrap you have to use **--ask-become-pass** parameters when running the playbook.
  ```
  ansible-playbook --ask-become-pass bootstrap.yml 
  ```

## Roles [14]
- Like below folder like structure we have to create roles.
- these main.yml file only contains the tasks part and are known as taskbook
```
roles/
├── base
│   └── tasks
│       └── main.yml
├── db_servers
│   └── tasks
│       └── main.yml
├── web_servers
│   ├── files
│   │   └── default_site.html
│   └── tasks
│       └── main.yml
└── workstations
    └── tasks
        └── main.yml
```
- if we are copying any files from the system to the targeted servers we have to place under the files directory inside these roles directories. like we have done for the web servers role.
- below code show how we have to mention these roles in the playbook.
```
- hosts: all
  become: true
  roles:
    - base
    
# Below plays will be ran in the servers which are in the web server group
- hosts: web_servers
  become: true
  roles:
    - web_servers


- hosts: db_servers
  become: true
  roles:
    - db_servers

#  installing the terraform in the local workstation setup
- hosts: workstations
  become: true
  roles:
    - workstations
```

## Host variables [15]
- We can use variables in the playbooks, previously we are passing the value of the variable in the inventory directory where ever we have given the ip address of the hosts.
- But this time we assign the values of the variable in the separate files for each hosts.
- directory name should be **host_vars**.
  - Inside this you have to create yaml files with the filename `ipaddress/dnsname/hostname`
  - like i am having the host `192.168.1.8` then the filename will be **192.168.1.8.yml**

### Using handlers 
- we can trigger a taskbook by using the **notify** module.
```
# changing the httpd configuration file
- name: change email address for admin
  tags: apache,centos,httpd
  lineinfile:
    path: /etc/httpd/conf/httpd.conf
    regexp: '^ServerAdmin'
    line: ServerAdmin sumesh@examplenew.com
  when: ansible_distribution == "CentOS"
  notify: restart_apache
```
- **handlers** directory has to be created inside `roles/webservers/handlers/main.yml`
```
- name: restart_apache
  tags: apache,httpd
  service:
    name: "{{ apache_service_name }}"
    state: restarted
```

## Templates 
- if we want to make changes to the configuration file and update that file in the servers.
- template file should be in format of `.j2` **jinja2**
- example we are going to create a template out of `sshd_config` file and will add  `AllowUsers` line with some users and inject this file in the host servers.
- under `roles/base/templates` i copy and pasted the sshd_config from the debian host and centos with the file extension **.j2**.
- then create two variables `ssh_allow_users` and `ssh_template_file` under **host_vars** directory.
```
ssh_allow_users: "osboxes simone"
ssh_template_file: ssh_debian_template.j2
```
- create a new task in the `roles/base/main.yml`
```
- name: allow users in ssh configuration
  template:
    src: "{{ ssh_template_file }}"
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: 0644
  notify: restart_ssh
```
- create a handlers 
```
- name: restart_ssh
  service:
    name: sshd
    state: restarted
```