---
layout: post
title:  "Ansible configure Domain Controller"
date:   2020-06-05
categories: [blog]
tags: [ansible, aws]
excerpt_separator: <!--more-->
---

So the journey to a 3 tier architecture on AWS using Packer, Terraform & Ansible begins. For reference read my earlier [post]({% post_url 2020-05-11-ansible-103 %}).\
The architecture will continue to evolve over a period of time & to keep it real, I thought of introducing a Windows Domain Controller.
After all, almost all the organizations have a DC for authentication & authorization.

<!--more-->

The CIs will be spun up using Terraform as will the entire infrastructure, including security groups, routing table, internet gateway, VPC etc. 
The configuration & management of the CIs will be done using Ansible.

I will share the code as I write them & in this post, I am sharing the code for configuring a Windows Domain Controller using Ansible.
As with any configuration management tool, there are more than one ways to write them including use of parameters. Feel free to adjust/modify as per your needs.

```yaml
---
  - name: Create New Active-Directory Domain & Forest
    hosts: localhost
    gather_facts: no

    vars_prompt:
        - name: domainname
          prompt: "Enter domain name"
    
    vars:
      safe_mode_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          39616164376432326663626231303866316663623561616263393734383331616632656264303362
          3532383365613036643664376639616139366364613331300a633037633537666537653333666665
          34306638376439663938653337343732653831626365376637653233316639396266616239653663
          6630633061623638380a656563653264366663333730656666616363356362366666376130323265
          64373835396164343739663932616163366432636361343637303638373933356539
      domain_name: {% raw %}"{{ domainname }}"{% endraw %}
      upstream_dns_1: 8.8.8.8
      upstream_dns_2: 8.8.4.4
      ntp_servers: "0.us.pool.ntp.org,1.us.pool.ntp.org,2.us.pool.ntp.org,3.us.pool.ntp.org"
      
    tasks:
      - name: Add host to Ansible inventory
        add_host:
          hostname: {% raw %}"{{ inventory_hostname }}"{% endraw %}
          ansible_connection: winrm
          ansible_winrm_transport: ntlm
          ansible_winrm_server_cert_validation: ignore
          ansible_winrm_port: 5985
      
      - name: Wait for system to become reachable over WinRM
        wait_for_connection:
          timeout: 900
        
      - name: Set upstream DNS server 
        win_dns_client:
          adapter_names: '*'
          ipv4_addresses:
          - {% raw %}"{{ upstream_dns_1 }}"{% endraw %}
          - {% raw %}"{{ upstream_dns_2 }}"{% endraw %}
      
      - name: Stop the time service
        win_service:
          name: w32time
          state: stopped
      
      - name: Set NTP Servers
        win_shell: 'w32tm /config /syncfromflags:manual /manualpeerlist:{% raw %}"{{ntp_servers}}"{% endraw %}'
      
      - name: Start the time service
        win_service:
          name: w32time
          state: started  
      
      - name: Disable firewall for Domain, Public and Private profiles
        win_firewall:
          state: disabled
          profiles:
          - Domain
          - Private
          - Public
        tags: disable_firewall
      
      - name: Install Active Directory domain services
        win_feature:
          name: AD-Domain-Services
          include_management_tools: yes
          include_sub_features: yes
          state: present
        register: domain_role
      - debug:
          msg: {% raw %}"{{ domain_role }}"{% endraw %}
      
      - name: Create new Windows domain in a new forest with specific parameters
        win_domain:
          create_dns_delegation: no
          database_path: C:\Windows\NTDS
          dns_domain_name: {% raw %}"{{ domain_name }}"{% endraw %}
          domain_mode: WinThreshold
          domain_netbios_name: {% raw %}"{{ domain_name.split('.')[0] | lower }}"{% endraw %}
          forest_mode: WinThreshold
          safe_mode_password: {% raw %}"{{ safe_mode_password }}"{% endraw %}
          sysvol_path: C:\Windows\SYSVOL
        register: ad
      
      - name: reboot server
        win_reboot:
          msg: "Installing Active Directory & Promoting to DC. Rebooting..."
          pre_reboot_delay: 15
        when: ad.changed
      
      - name: Set internal DNS server 
        win_dns_client:
          adapter_names: '*'
          ipv4_addresses:
          - '127.0.0.1'
      
      - name: ensure ADWS service is started
        win_service:
          name: ADWS
          state: started
        register: adws_service
      - debug:
          msg: {% raw %}"{{ adws_service }}"{% endraw %}
```

I am using string encryption for encrypting safe mode password, I have linked documentation on [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html) which explains the process in detail.

In the next post, AWS SSM Session Manager.

Stay Safe!