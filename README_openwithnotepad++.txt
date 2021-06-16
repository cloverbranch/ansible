------------------------------------------------------------------------------------
|Documentation for deploying httpd webserver with ssl on ec2 instance using ansible|
------------------------------------------------------------------------------------

+++INSTALLING NECCESSARY PACKAGES FOR ANSIBLE SERVER+++
-->yum install python python-pip python-setuptools*
-->curl -O https://bootstrap.pypa.io/get-pip.py or curl -O https://bootstrap.pypa.io/pip/2.7/get-pip.py
-->python get-pip.py
-->pip install boto
-->pip install boto3
-->yum install ansible (for centos) or amazon-linux-extras install ansible2 (amazon-ami)

******************************************************************************************************
once the packages installed , add your awskeys(IAM user) permanently on server using vi editor 
-->vi ~/.boto 
[Credentials]
aws_access_key_id = your_key(change_me)
aws_secret_access_key = your_secret_key(change_me)

******************************************************************************************************
Now edit the ansible.cfg file ,so that when ec2 spins up it will take neccessary permission and ssh key for it 
[defaults]
host_key_checking = false
remote_user = ec2-user
ask_pass = false
private_key_file = /your_path_to_key/awssshkey.pem
[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false 

******************************************************************************************************
Now upto this point everything is setup on server , now we start writing our files 
we make to file one is ==>infra.yml<==(main file path :-/etc/ansible/) ++ ==>main.yml<==(role file for httpdsetup path:- /etc/ansible/roles/httpdsetup/tasks/)
if roles/httpdsetup/tasks/ directory is not present make manually 

                                         INFRA.YML
-------------------------------------------------------------------------------------------------------------
---                                                                                                         |
  - name: Provision an EC2 Instance                                                                         |
    hosts: localhost                                                                                        |
    connection: local                                                                                       |--------> first we define localhost in hosts which
    gather_facts: False                                                                                     |          which is connecting local
    tags: provisioning                                                                                      |
                                                                                                            |
    vars:                                                                                                   |
      instance_type: t2.micro                                                                               |--------> now we are defining variables for the 
      security_group: webserver                                                                             |          what type of instance we need,security 
      image: ami-0742b4e673072066f                                                                          |          group ,ami image id,region ,sshkey name
      region: us-east-1                                                                                     |          and count for number of instances to launch
      keypair: awssshkey                                                                                    |
      count: 1                                                                                              |
                                                                                                            |
    tasks:                                                                                                  |
                                                                                                            |--------> now in tasks section we first use ec2_group
      - name: Create New security group with below given name                                               |          module to create security group which open
        local_action:                                                                                       |          port 22 and 443 and reject all other connections
          module: ec2_group                                                                                 |
          name: "{{ security_group }}"                                                                      |
          description: Security Group for Newly Created EC2 Instance                                        |
          region: "{{ region }}"                                                                            |
          rules:                                                                                            |
            - proto: tcp                                                                                    |
              from_port: 22                                                                                 |
              to_port: 22                                                                                   |
              cidr_ip: 0.0.0.0/0                                                                            |
            - proto: tcp                                                                                    |
              from_port: 443                                                                                |
              to_port: 443                                                                                  |
              cidr_ip: 0.0.0.0/0                                                                            |
          rules_egress:                                                                                     |
            - proto: all                                                                                    |
              cidr_ip: 0.0.0.0/0                                                                            |
                                                                                                            |
                                                                                                            |
      - name: Launch the new t2 micro EC2 Instance                                                          |---------> now use module ec2 and pass the variables
        local_action: ec2                                                                                   |           in given fields and register data on variable
                      group={{ security_group }}                                                            |           ec2 which we use to do ssh and make a temp
                      instance_type={{ instance_type}}                                                      |           inventory.
                      image={{ image }}                                                                     |
                      wait=true                                                                             |
                      region={{ region }}                                                                   |
                      keypair={{ keypair }}                                                                 |
                      count={{count}}                                                                       |
                      instances_tags:                                                                       |
                        Name: launched_servers                                                              |
        register: ec2                                                                                       |
                                                                                                            |
      - name: Wait for SSH access                                                                           |---------> we check for ssh now to come up using wait_for
        local_action: wait_for                                                                              |           module ,in {{ ec2.instances }} ec2 is registered
                      host={{ item.public_ip }}                                                             |           variable from above and instances is the attribute
                      port=22                                                                               |
                      state=started                                                                         |
        with_items: "{{ ec2.instances }}"                                                                   |
                                                                                                            |
      - name: adding server to temp group                                                                   |---------> this is the dynamic inventory we created 
        add_host:                                                                                           |           using add_host module and give groupname 
          hostname: "{{ item.public_ip}}"                                                                   |           to it.
          groupname: launched_server                                                                        |
        with_items: "{{ ec2.instances }}"                                                                   |
                                                                                                            |
                                                                                                            |
  - name: setting up httpd webserver                                                                        |---------> now we use dynamic inventory using launched_server
    hosts: launched_server                                                                                  |           host name we created above in groupname and which execute
    roles:                                                                                                  |           /etc/ansible/roles/httpdsetup/tasks/main.yml using roles
      - httpdsetup                                                                                          |
-------------------------------------------------------------------------------------------------------------

********************************************************************************************************

Now we create role file for httpd setup
                                      MAIN.YML
-------------------------------------------------------------------------------------------------------------
---                                                                                                         |
   - name: installing httpd and mod_ssl                                                                     |
     yum:                                                                                                   |--------> using yum module first we install httpd
      name: "{{ item }}"                                                                                    |          mod_ssl and git package on lauched server
      state: installed                                                                                      |          and starting httpd service.
     with_items:                                                                                            |
       - httpd                                                                                              |
       - git                                                                                                |
       - mod_ssl                                                                                            |
   - name: starting service                                                                                 |
     service:                                                                                               |
      name: httpd                                                                                           |
      state: started                                                                                        |
   - name: Clone git to a local server                                                                      |
     git:                                                                                                   |
       repo: https://github.com/cloverbranch/ansible.git                                                    |--------> now we clone from git to lauched server
       dest: "/var/www/html/mysite"                                                                         |          
   - name: Ansible create file if it doesn't exist example                                                  |
     file:                                                                                                  |
       path: "/etc/httpd/conf.d/web02.conf"                                                                 |--------> now we create httpd conf 
       state: touch                                                                                         |
   - name: genrate private key                                                                              |
     openssl_privatekey: path=/etc/ssl/certs/fatty.key                                                      |--------> genrating key,csr,and using both file 
   - name: genrate csr                                                                                      |          to create certificate file
     openssl_csr:                                                                                           |
          path: /etc/ssl/certs/fatty.csr                                                                    |
          privatekey_path: /etc/ssl/certs/fatty.key                                                         |
          common_name: redhatme                                                                             |
          country_name: IN                                                                                  |
          organization_name: Demo                                                                           |
   - name: Genrate a self signed certificate                                                                |
     openssl_certificate:                                                                                   |
          csr_path: /etc/ssl/certs/fatty.csr                                                                |
          path: /etc/ssl/certs/fatty.crt                                                                    |
          privatekey_path: /etc/ssl/certs/fatty.key                                                         |
          provider: selfsigned                                                                              |
   - name: Ansible Insert multiple lines using blockinfile                                                  |
     blockinfile:                                                                                           |--------> now edit httpd  conf file we created above using 
       dest: /etc/httpd/conf.d/web02.conf                                                                   |          blockinfile module
       block: |                                                                                             |
         <virtualHost localhost:443>                                                                        |
         serverAdmin root@localhost                                                                         |
         serverName localhost                                                                               |
         DocumentRoot /var/www/html                                                                         |
         DirectoryIndex index.html                                                                          |
         SSLEngine on                                                                                       |
         SSLCertificatefile /etc/ssl/certs/fatty.crt                                                        |
         SSLCertificateKeyfile /etc/ssl/certs/fatty.key                                                     |
         </virtualHost>                                                                                     |
         <Directory /var/www/html>                                                                          |
         Require all granted                                                                                |
         </Directory>                                                                                       |
       backup: no                                                                                           |-------> restarting httpd service 
   - name: restarting service                                                                               |
     service:                                                                                               |
       name: httpd                                                                                          |
       state: restarted                                                                                     |
-------------------------------------------------------------------------------------------------------------

NOW CONNECT TO
https://lauchedserverip/mysite

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++THANKS+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++