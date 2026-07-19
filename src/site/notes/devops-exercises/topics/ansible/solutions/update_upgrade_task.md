---
{"dg-publish":true,"permalink":"/devops-exercises/topics/ansible/solutions/update_upgrade_task/","dg-note-properties":{}}
---


## Update and Upgrade apt packages task - Solution

```
- name: "update and upgrade apt packages."
  become: yes
  apt:
    upgrade: yes
    update_cache: yes
```
