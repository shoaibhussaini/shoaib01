---
- name: Dummy Playbook for Validation
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Output a debug message
      debug:
        msg: "The playbook has been called successfully!"
