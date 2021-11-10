---
layout: post
title:  "Configure Windows Instance Using Ansible"
date:   2020-06-23
categories: [blog]
tags: [aws, ansible, bash]
excerpt_separator: <!--more-->
---

Carrying forward from my last [post]({% post_url 2020-06-16-hashicorp-packer-windowsami %}) where an image was built using Packer, in this post I will show how to configure a Windows instance using Ansible.\
This is going to be a basic configuration Playbook which can be used as a building template. Some of the play can be modified, completely removed or new plays can be added.\
I hope this Playbook will give everyone something for their own use. 

<!--more-->

```yaml{% raw %}
---
  - name: Configure Windows Server (renaming, cleanup, adding Ansible user)
    hosts: all
    gather_facts: yes
    vars_prompt: 
      - name: username
        prompt: "Enter local username"
        private: no
      - name: password
        prompt: "Enter password"
    vars: 
      ansible_user: "{{ username }}"
      ansible_password: "{{ password }}"
      ansible_connection: winrm
      ansible_winrm_port: 5985
      ansible_winrm_server_cert_validation: ignore
      
    tasks:
      - name: Change the hostname
        win_hostname:
          name="{{ inventory_hostname }}" # Magic Variable
        register: res
      - debug:
          var: res

      - name: Uptime before reboot
        win_shell: |
          get-ciminstance win32_operatingsystem | select-object lastbootuptime
        register: uptime_pre_reboot
      - debug:
          msg: "{{ uptime_pre_reboot.stdout_lines }}"

      - name: Reboot
        win_reboot:
        when: res.reboot_required
      
      - name: Wait for {{ inventory_hostname }} to come back up
        wait_for_connection:
          delay: 60
          timeout: 120
        when: res.reboot_required
      
      - name: Uptime after reboot
        win_shell: |
          get-ciminstance win32_operatingsystem | select-object lastbootuptime
        register: uptime_post_reboot
      - debug:
          msg: "{{ uptime_post_reboot.stdout_lines }}"

      # gathering the facts again for assertion task
      - name: do facts module to get latest information
        setup:

      - name: Validate ansible_fqdn == inventory_hostname
        tags:
          - validate
        assert:
          that:
            ansible_fqdn == inventory_hostname
      
      - name: cleaning up Temp folder
        win_shell: |
          $tempFiles = Test-Path -Path C:\windows\temp
          if ($tempFiles) {
            Remove-Item -Path C:\Windows\temp\*.ps1 -Recurse -Force
          }

      - name: adding Ansible local user
        win_user:
          name: ansible
          password: "{{ password }}"
          state: present
          groups:
            - Administrators
      
      - name: installing SSM plugin
        win_package: 
          path: https://s3.amazonaws.com/session-manager-downloads/plugin/latest/windows/SessionManagerPluginSetup.exe
          product_id: '{2A807C76-98F0-48F2-A763-1AB5756E479B}'
          arguments:
          - /S
        register: ssm_install
      - debug:
          var: ssm_install
```{% endraw %}
Here’s the breakdown of the Playbook;
The `vars_prompt` section is prompting for local Administrator account and a Password. 
Since this is a sample template and I have configured the image with a local administrative account. Those preconfigured Administrator credentials is what I am obtaining here.
NOTE --> You can customize this section or totally remove it depending on your needs. 
The `vars` block can be configured in `/etc/ansible/hosts` file but I have also put it here in the Playbook.
The first Play is checking for `hostname` which should match the `inventory_hostname`, something that is set in `/etc/ansible/hosts` file, below is a sample.
```bash
[win]
win1 ansible_host=172.16.X.X 
```
Further reading on Ansible [magic variable]( https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)

The first `UPTIME` block is for recording time before system goes for a reboot. I put it there as a check as changing hostname requires a reboot and I wanted to see that in action.\
The `REBOOT` block runs on condition match `when: res.reboot_required` and waits for system to come back up.\
Second `UPTIME` prints out time after the reboot, just an extra check.\
In the next block, I am gathering facts again for validating assertion that `hostname` has actually changed.\
Working with Packer, I noticed sensitive residual stored in `C:\Windows\Temp` folder which this block will clean up.\
I am adding a local user to the Windows server and making it part of Administrator group.\
Installing `AWS SSM optional plugin` for initiating remote session to Windows instances without having to RDP in.  Something that I have covered in one of my previous [post]({% post_url 2020-06-08-aws-ssm-startsession %})

Here I am running the Playbook against a freshly spun up Windows instance.

```bash
[OpsToDevops@localhost]# ansible-playbook configure_windows.yml --limit win1
Enter local username: Administrator
Enter password: 

PLAY [Configure Windows Server (renaming, cleanup, adding Ansible user)] *******************************************************************

TASK [Gathering Facts] *********************************************************************************************************************
ok: [win1]

TASK [Change the hostname] *****************************************************************************************************************
ok: [win1]

TASK [debug] *******************************************************************************************************************************
ok: [win1] => {
    "res": {
        "changed": false, 
        "failed": false, 
        "old_name": "EC2AMAZ-SKTRH2T", 
        "reboot_required": true
    }
}

TASK [Uptime before reboot] ****************************************************************************************************************
changed: [win1]

TASK [debug] *******************************************************************************************************************************
ok: [win1] => {
    "msg": [
        "", 
        "lastbootuptime      ", 
        "--------------      ", 
        "6/23/2020 1:36:12 AM", 
        "", 
        ""
    ]
}

TASK [Reboot] ******************************************************************************************************************************
changed: [win1]

TASK [Wait for win1 to come back up] ******************************************************************************************************
ok: [win1]

TASK [Uptime after reboot] *****************************************************************************************************************
changed: [win1]

TASK [debug] *******************************************************************************************************************************
ok: [win1] => {
    "msg": [
        "", 
        "lastbootuptime      ", 
        "--------------      ", 
        "6/23/2020 1:36:12 AM", 
        "", 
        ""
    ]
}

TASK [do facts module to get latest information] *******************************************************************************************
ok: [win1]

TASK [Validate ansible_fqdn == inventory_hostname] *****************************************************************************************
ok: [win1] => {
    "changed": false, 
    "msg": "All assertions passed"
}

TASK [cleaning up Temp folder] *************************************************************************************************************
changed: [win1]

TASK [adding Ansible local user] ***********************************************************************************************************
ok: [win1]

TASK [installing SSM plugin] ***************************************************************************************************************
ok: [win1]

TASK [debug] *******************************************************************************************************************************
ok: [win1] => {
    "ssm_install": {
        "changed": false, 
        "failed": false, 
        "reboot_required": false
    }
}

PLAY RECAP *********************************************************************************************************************************
win1                      : ok=14   changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


```

The main concept behind CM tools is Idempotency. For the uninitiated, Idempotence is the property of certain operations in mathematics and computer science whereby an operation can be applied multiple times but the result will NOT change.\
Example, take 1, add 1 to it, the result will still be 1 & NOT 2, again add 1, still 1.\
Now taking that logic of Idempotence, what do you think will happen, if this Playbook is run the second time against the same Windows instance, or third or tenth time?\
I hope that this Playbook will have something for everyone, whether starting out with Plays or improving on current Playbooks.

Until next time, keep on learning and stay safe!
