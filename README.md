
This demonstrates that https://github.com/ansible/ansible/issues/56046 isn't yet fixed, contrary to https://github.com/ansible/ansible/issues/56046#issuecomment-889418109.

This repo has these inter-dependencies:

```
$ cat site.yml
- import_playbook: base.yml
- import_playbook: monitor.yml
$ egrep '*' roles/*/meta/*.yml
roles/base/meta/main.yml:dependencies:
roles/base/meta/main.yml:  - role: upgrades
roles/base/meta/main.yml:  - role: mailer
roles/monitor/meta/main.yml:---
roles/monitor/meta/main.yml:dependencies:
roles/monitor/meta/main.yml:  #- role: netdata # in the real repo, we have a role to install netdata
roles/monitor/meta/main.yml:  - role: mailer   # and we install 'mailer' so it can send us notifications; they aren't mandatorily enabled by the netdata role
roles/upgrades/meta/main.yml:dependencies:
roles/upgrades/meta/main.yml:  - role: mailer
roles/upgrades/meta/main.yml:  #- role: jnv.unattended-upgrades # in my full repo, I'm using this ansible-galaxy role
```

or, more schematically

```
site -> base
site -> monitor

base -> upgrades
base -> mailer

monitor -> mailer

upgrades -> mailer
```


When I run this, I get

```
$ ansible-playbook --version
ansible-playbook 2.10.8
  config file = /home/kousu/src/neuropoly/reproducer/ansible.cfg
  configured module search path = ['/home/kousu/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/kousu/.local/lib/python3.6/site-packages/ansible
  executable location = /home/kousu/.local/bin/ansible-playbook
  python version = 3.6.9 (default, Jan 26 2021, 15:33:00) [GCC 8.4.0]

$ ansible-playbook site.yml

PLAY [base] ******************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************
ok: [localhost]

TASK [mailer : debug] ********************************************************************************************************************************
ok: [localhost] => {
    "msg": "This is roles/mailer running at 1637879040. According to https://github.com/s-hertel/ansible/blob/cb44698afdc68b605950b0ce69aeae746fdc30cc/docs/docsite/rst/user_guide/playbooks_reuse_roles.rst#running-role-dependencies-multiple-times-in-one-playbook, I should only run *once*."
}

TASK [mailer : Install mail software] ****************************************************************************************************************
ok: [localhost] => {
    "msg": "apt: name: - opensmtpd - mailutils state: present"
}

TASK [mailer : Forward mail to sysadmins] ************************************************************************************************************
ok: [localhost]

TASK [base : software] *******************************************************************************************************************************
ok: [localhost] => {
    "msg": "apt: name: - sudo - etckeeper - ethtool state: present install_recommends: no"
}

TASK [base : Replace a localhost entry with our own] *************************************************************************************************
ok: [localhost]

TASK [base : Timezone] *******************************************************************************************************************************
ok: [localhost]

PLAY [localhost] *************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************
ok: [localhost]

TASK [mailer : debug] ********************************************************************************************************************************
ok: [localhost] => {
    "msg": "This is roles/mailer running at 1637879046. According to https://github.com/s-hertel/ansible/blob/cb44698afdc68b605950b0ce69aeae746fdc30cc/docs/docsite/rst/user_guide/playbooks_reuse_roles.rst#running-role-dependencies-multiple-times-in-one-playbook, I should only run *once*."
}

TASK [mailer : Install mail software] ****************************************************************************************************************
ok: [localhost] => {
    "msg": "apt: name: - opensmtpd - mailutils state: present"
}

TASK [mailer : Forward mail to sysadmins] ************************************************************************************************************
ok: [localhost]

TASK [monitor : Install nginx] ***********************************************************************************************************************
ok: [localhost] => {
    "msg": "apt: name: - nginx state: present"
}

PLAY RECAP *******************************************************************************************************************************************
localhost                  : ok=12   changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

That is, the line that "should only run *once*" actually runs twice. Which is actually even more confusing because it's included three times, so the de-duplication logic seems to be *partially* working.
