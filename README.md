# ansible-tags-test
test strange tags behavior on ansible

##playbook
to demonstrate that if a first (pre-run) task sets `gather_facts` to `false`,
subsequent calls to rolles will skip gathering facts if we use a tag just applying to a role.

playbook
```yaml
---
- name: pre ...
  hosts: all
  gather_facts: "{{ pre_gather_facts | default(false) | bool }}"
  # pre_tasks:
  tasks:
    - name: ansible version
      delegate_to: 127.0.0.1
      debug:
        var: ansible_version.full
  tags: always

- name: test facts and tags
  hosts: all
  gather_facts: yes
  roles:
    - role: testfacts
      tags:
        - innertag
  tags:
    - outertag
```

If called with the `outertag`, facts are gatered as expected:
```
± ansible-playbook  -i hosts -t outertag playbook.yml

PLAY [pre ...] ****************************************************************************************************************************************************************************************************

TASK [ansible version] ********************************************************************************************************************************************************************************************
ok: [127.0.0.1] => {
    "ansible_version.full": "2.12.2"
}

PLAY [test facts and tags] ****************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************
ok: [127.0.0.1]

TASK [testfacts : debug fact ansible_distribution] ****************************************************************************************************************************************************************
ok: [127.0.0.1] => {
    "ansible_distribution": "Debian"
}

TASK [testfacts : assert fact ansible_distribution] ***************************************************************************************************************************************************************
ok: [127.0.0.1] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY RECAP ********************************************************************************************************************************************************************************************************
127.0.0.1                  : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

but for the `innertag`, facts are not gathered
```
± ansible-playbook  -i hosts -t innertag playbook.yml

PLAY [pre ...] ****************************************************************************************************************************************************************************************************

TASK [ansible version] ********************************************************************************************************************************************************************************************
ok: [127.0.0.1] => {
    "ansible_version.full": "2.12.2"
}

PLAY [test facts and tags] ****************************************************************************************************************************************************************************************

TASK [testfacts : debug fact ansible_distribution] ****************************************************************************************************************************************************************
ok: [127.0.0.1] => {
    "ansible_distribution": "VARIABLE IS NOT DEFINED!"
}

TASK [testfacts : assert fact ansible_distribution] ***************************************************************************************************************************************************************
fatal: [127.0.0.1]: FAILED! => {
    "assertion": "ansible_distribution is defined",
    "changed": false,
    "evaluated_to": false,
    "msg": "Assertion failed"
}

PLAY RECAP ********************************************************************************************************************************************************************************************************
127.0.0.1                  : ok=2    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

