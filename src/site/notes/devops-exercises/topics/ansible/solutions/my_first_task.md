---
{"dg-publish":true,"permalink":"/devops-exercises/topics/ansible/solutions/my_first_task/","dg-note-properties":{}}
---


## My First Task - Solution

```
- name: Create a new directory
  file:
    path: "/tmp/new_directory"
    state: directory
```
