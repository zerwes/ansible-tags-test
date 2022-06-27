# ansible-tags-test
test and demonstarte strange tags behavior on ansible regarding `Gathering Facts`

see: https://github.com/ansible/ansible/issues/78153

As this is documented and the behavior is as expected,
I am archiving this (even if the logic and behavior is not comprehensible for me).

## playbook
playbook to demonstrate that if a first (pre-run) task sets `gather_facts` to `false`,
subsequent calls to roles will skip gathering facts if we use a tag just applying to a role.

playbook:
```yaml
---
- name: pre ...
  hosts: all
  gather_facts: no
  # pre_tasks:
  tasks:
    - name: ansible version
      delegate_to: 127.0.0.1
      debug:
        var: ansible_version.full
  tags: always

# my solution for now
- name: ensure facts gathering is done and not disturbed by some butterfly wing noise on the other side of the play
  hosts: all
  pre_tasks:
    - name: gather facts
      setup:
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

## ansible versions
tested with:

  * 2.9.27
  * 2.12.2
  * 2.12.7
  * 2.13.1
