# AWS EC2 SSH Key Rotation

Sometimes we get the requirement to change the key-pair of AWS EC2 instances for some security reasons. In this article, we will be changing the key pair of running EC2 instances using Ansible Playbook.

## General Information

When you use standard AMIs to launch an EC2 instance, you can connect to it using remote access protocols like SSH and RDP. AWS provides asymmetric key pairs called Amazon EC2 key pairs that you can use to authenticate yourself to the EC2 instance.
For encrypting and decrypting login information, EC2 Key Pairs are used: Public Key and Private Key. Public–key cryptography encrypts any piece of data, such as a password, with a public key, and then the receiver decrypts the data with a private key.
To connect to your instance, you must first generate a key pair, identify the name of that key pair when the instance is launched, and provide information about the private key when connecting.
As a AWS security best practice, it is necessary to regularly rotate EC2 key pairs within your account. 

## Technology Used
- ansible - version 2.9.20

## Prerequisites
- Ansible version - 2.9
- Ansible Master Server - Linux
- SSH access to client servers
- SSH PEM key of the client servers
- SSH User with sudo privilege
- AWS Access key and Secret Key

## Features

- SSH key will be changed retaining same key pair name.
- No need to mention any hosts (Inventory) as we are using Dynamic Inventory function of Ansible in this playbook.
- Rotation of key pairs for multiple instances can be done in a single session.
- Free: Ansible is an open-source tool.
- Very simple to set up and use: No special coding skills are necessary to use Ansible’s playbooks.

## About the playbook

The playbook will perform following operations;
- Gather Information of the EC2 instances in which Key To Be Rotated
- Create Inventory Of EC2 With Old SSH-keyPair
- Create New SSH-Key Material
- Add New SSH Public Key to authorized_key
- Check SSH Connectivity To EC2 instance Using Newly Added Key
- Execute the Uptime command on remote servers
- Remove Old SSH Public Key and add New SSH Public Key to authorized_key
- Print Old authorized_keys file
- Print New authorized_keys file
- Replace Old SSH public key From AWS Account
- Rename new Private Key Locally in ansible master server
- Rename Public Key locally in ansible master server

The playbook has been written below::
~~~
---
- name: "Creation of the Ansible Inventory Of EC2 Instances in which Key To Be Rotated"
  hosts: localhost
  vars_files:
    - key.vars
  tasks:

    # ---------------------------------------------------------------
    # Getting Information of the EC2 instances in which Key To Be Rotated
    # ---------------------------------------------------------------

    - name: "Fetching Details About EC2 Instance"
      ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          "key-name": "{{ key_name }}"
          instance-state-name: [ "running" ]
      register: ec2


    # ------------------------------------------------------------
    # Creating Inventory Of EC2 With Old SSH-keyPair
    # ------------------------------------------------------------
    - name: "Creating Inventory "
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "aws"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: "{{ ssh_port }}"
        ansible_user: "{{ system_user }}"
        ansible_ssh_private_key_file: "{{ key_name }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ ec2.instances }}"
      no_log: true

- name: "Updating SSH-Key Material"
  hosts: aws
  become: true
  gather_facts: false
  vars_files:
    - key.vars

  tasks:

    - name: "Register current SSH Authorized_key file of the system user"
      shell: cat /home/"{{system_user}}"/.ssh/authorized_keys
      register: oldauth

    - name: "Creating New SSH-Key Material"
      delegate_to: localhost
      run_once: True
      openssh_keypair:
        path: "key_tmp"
        type: rsa
        size: 4096
        state: present

    - name: "Adding New SSH-Key Material"
      authorized_key:
        user: "{{ system_user }}"
        state: present
        key: "{{ lookup('file', 'key_tmp.pub')  }}"


    - name: "Creating SSH Connection Command"
      set_fact:
        ssh_connection: "ssh -o StrictHostKeyChecking=no -i key_tmp {{ ansible_user }}@{{ ansible_host }} -p {{ ansible_port }} 'uptime'"


    - name: "Checking Connectivity To EC2 Using Newly Added Key"
      ignore_errors: true
      delegate_to: localhost
      shell: "{{ ssh_connection }}"

    - name: "Executing the Uptime command on remote servers"
      command: "uptime"
      register: uptimeoutput
    - debug:
        var: uptimeoutput.stdout_lines

    - name: "Removing Old SSH Public Key and adding New SSH Public Key to authorized_key"
      authorized_key:
        user: "{{ system_user }}"
        state: present
        key: "{{ lookup('file', 'key_tmp.pub')  }}"
        exclusive: true
    

    - name: "Print Old authorized_keys file"
      debug:
        msg: "SSH Public Keys in Old authorized_keys file are '{{ oldauth.stdout }}'"


    - name: "Print New authorized_keys file"
      shell: cat /home/"{{system_user}}"/.ssh/authorized_keys
      register: newauth
    - debug:
        msg: "SSH Public Keys in New authorized_keys file are '{{ newauth.stdout }}'"


    - name: "Replacing Old SSH public key From AWS Account"
      delegate_to: localhost
      run_once: True
      ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ key_name }}"
        key_material: "{{ lookup('file', 'key_tmp.pub') }}"
        force: true
        state: present

    - name: "Renaming new Private Key Locally"
      delegate_to: localhost
      run_once: True
      shell: |
        mv key_tmp {{ key_name }}.pem
        chmod 400 {{ key_name }}.pem

    - name: "Renaming Local Public Key"
      run_once: True
      delegate_to: localhost
      shell: "mv key_tmp.pub  {{ key_name }}.pub"
~~~

## Setup

### Installation of Ansbile and Boto (In Ubuntu)

~~~sh
- apt-get update
- apt-get install python3
- apt-get install python3-pip
- pip3 install ansible
- pip3 install boto3 botocore
~~~
![alt text](https://i.ibb.co/bdBtdhk/1.png)

## Usage

##### 1. Pull the code from github repository

~~~
git clone https://github.com/sebinxavi/aws-key-rotation-without-changing-keyname.git
cd aws-key-rotation-without-changing-keyname
~~~
![alt text](https://i.ibb.co/gdfSfxY/Capture.png)

##### 2. Update the Ansible variable file - key.vars
~~~
access_key: "<>"
secret_key: "<>"
region: "<>"  #----> Example: "ap-south-1"
key_name: "<>"   #----> Upload this Pem file in the same directory with 400 Permission.
system_user: "<>"
ssh_port: 22
~~~

access_key: Add your AWS access key.
secret_key: Add your AWS secret key.
region: Add the AWS region in which EC2 instances are hosted. Example, "ap-south-1"
key_name: Add your SSH key name and upload the SSH private key in the same location with 400 permission
system_user: Add the System user
ssh_port: Add the SSH port number

![alt text](https://i.ibb.co/r2vfgnF/2.png)

##### 3. Run the Ansible playbook
~~~
ansible-playbook main.yml
~~~

![alt text](https://i.ibb.co/YRg3VH7/3.png)

Check the SSH Public Key listed in authorised_key file
![alt text](https://i.ibb.co/gMRpRpx/4.png)

##### 4. Try to SSH to Server using the updated the SSH key
~~~
ssh -i server-key.pem user@{server-IP}}
~~~
![alt text](https://i.ibb.co/4dvjR8S/5.png)

## Further Information

In this playbook, We are using the same SSH key pair name for accessing the server with replaced SSH key material. I have created another ansible playbook that can be used if you would like to access the server with new key name. If you would like to access the server with new SSH key name, please refer another playbook from my [Github repository](https://github.com/sebinxavi/aws-key-rotation.git)

## Author
Created by [@sebinxavi](https://www.linkedin.com/in/sebinxavi/) - feel free to contact me and advise as necessary!

<a href="mailto:sebin.xavi1@gmail.com"><img src="https://img.shields.io/badge/-sebin.xavi1@gmail.com-D14836?style=flat&logo=Gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/sebinxavi"><img src="https://img.shields.io/badge/-Linkedin-blue"/></a>
