## SSH-Key Rotation Using Ansible

Here is a project to change the key material of a currenlty active ssh key pair. The job describes how we find the instances that are using the key wheich we need to changed and how to change the key material with new one using Ansible.

#### Features

1. Easy to change the key pair on all the intances which is built on it. 
2. The key name will be same, but the material will be changed
3.  We are using dynamic inventory, so no need to have predefined inventory file 


---
## Pre-Requests 
- Install Ansible on your Master Machine
- Create an IAM user  on your AWS account and use it while executing the playbook
- 
##### Installations
[Ansible2](https://docs.ansible.com/ansible/2.3/index.html) (For your reference visit [How to install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html))
##### IAM Role Creation
[IAM Role Creation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create.html)
##### Ansible Modules used
- [ec2_instance_info](https://docs.ansible.com/ansible/latest/collections/community/aws/ec2_instance_info_module.html) 
- [ec2-key](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_key_module.html)
- [openssh_keypair](https://docs.ansible.com/ansible/latest/collections/community/crypto/openssh_keypair_module.html)
- [authorized_key](https://docs.ansible.com/ansible/2.4/authorized_key_module.html)
- [shell](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)
- ---

Let's see how the process flow is 

![alt text](https://i.ibb.co/Y33DshH/Screenshot.png)

Here is how the code goes
```sh
---
- name: " Playbook for key rotation"
  hosts: localhost
  become: true
  vars_files:
  - access_key.vars
  tasks:  
    # ---------------------------------------------------------------
    # Getting Information About Of Ec2's Which Need key To be Rotated
    # ---------------------------------------------------------------
    - name:
    ec2_instance_info:
      aws_access_key: "{{ access_key }}"
      aws_secret_key: "{{ secret_key }}"
      region: "{{region}}"
      filters:
        "key-name": "{{ key_name }}"
        instance-state-name: [ "running"]
    register: ec2_info
    # ------------------------------------------------------------
    # Creating Inventory Of Ec2  With Old Ssh-keyPair
    # ------------------------------------------------------------    
  - name: creating inventory
    add_host:
      name: "{{item.private_ip_address}}"
      groups: "rotation"
      ansible_host: "{{ item.private_ip_address }}"
      ansible_port: 22
      ansible_user: "ec2-user"
      ansible_ssh_private_key_file: "{{ key_name }}.pem"
      ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
    with_items:
    - "{{ ec2_info.instances }}"

    # ---------------------------------------------------------------
    # Changing The Key Pair Process
    # ---------------------------------------------------------------
   
- name: SSH key updation
  hosts: rotation
  become: true
  vars_files:
  - access_key.vars
  tasks:
    # ---------------------------------------------------------------
    # Creating New Key Pair
    # ---------------------------------------------------------------

  - name: creating new key pair
    delegate_to: localhost
    run_once: true
    openssh_keypair:
      path: "{{key_tmp}}"
      type: rsa
      size: 4096
      state: present 

  - name: Adding New SshKey Meterial to Authorized keys
    authorized_key:
      user: ec2-user
      state: present
      key: "{{ lookup('file', '{{ key_tmp }}.pub')  }}"

    # ---------------------------------------------------------------
    # Checking whether we are able to SSH into server with new key
    # ---------------------------------------------------------------

  - name: "Creating Ssh Connection Command"
    set_fact:
      ssh_connection: "ssh -o StrictHostKeyChecking=no -i {{ key_tmp }} {{ansible_ssh_user}}@{{ ansible_ssh_host }} 'uptime'"

  - name: "checking the ssh connection with newly added key"
    delegate_to: localhost
    shell: "{{ ssh_connection }}"
    register: ssh_check
    # ---------------------------------------------------------------
    # Removing old Key material
    # ---------------------------------------------------------------
  - name: "Removing Old KeyMeterial"
    when: ssh_check.changed == true
    authorized_key:
      user: ec2-user
      state: present
      key: "{{ lookup('file', '{{ key_tmp }}.pub')  }}"
      exclusive: true 
    # ---------------------------------------------------------------
    # Updating the new key material to AWS account
    # ---------------------------------------------------------------

  - name: Changing key material on aws account
    delegate_to: localhost
    run_once: True
    ec2_key:
      aws_access_key: "{{ access_key }}"
      aws_secret_key: "{{ secret_key }}"
      region: "{{region}}"
      name: "{{key_name}}"
      key_material: "{{ lookup('file', '{{ key_tmp }}.pub') }}"
      force: true
      state: present
    # ---------------------------------------------------------------
    # Replacing the local key material 
    # ---------------------------------------------------------------

  - name: changing local public key
    delegate_to: localhost
    run_once: true
    shell: "mv {{key_tmp}}.pub {{key_name}}.pub"
  
  - name: changing local private key
    delegate_to: localhost
    run_once: true
    shell: "mv {{key_tmp}} {{key_name}}.pem"  
```
Let's dig deeper into the code
```sh
 - name: creating new key pair
    delegate_to: localhost
    run_once: true
    openssh_keypair:
      path: "{{key_tmp}}"
      type: rsa
      size: 4096
      state: present 
```

> delegate_to : localhost
This step is to run the code on the host machine so that we can copy to all the intances afterwards.

> run_once: true
If not set, the  task will execute as many time as the number of instances that are in the inventory file. We just require to run ince as we are running it on the local machine. 

```sh
  - name: "Creating Ssh Connection Command"
    set_fact:
      ssh_connection: "ssh -o StrictHostKeyChecking=no -i {{ key_tmp }} {{ansible_ssh_user}}@{{ ansible_ssh_host }} 'uptime'"

  - name: "checking the ssh connection with newly added key"
    delegate_to: localhost
    shell: "{{ ssh_connection }}"
    register: ssh_check
```
> set_fact:  this is used to create dynamic variable during execution. 
ansible_ssh_user, ansible_ssh_host are the ansible's magic variable/Special variables. You can check it  [here](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)

Once it succeeded, we can change the key file on the instances and also the key material from AWS account. 
Laste step is to change our newly created key on our local machin to old key name

That's it

------------------------------- Gigin George ----------------------


----------------------------- gigingkallumkal@gmail.com -----------




