This WIP allows configuring restic to use password or password file.
New variables are introduced:
`restic_use_password_file` - boolean, activates tasks for managing password file
`restic_password_dest` - string ,sets directory where file will be created. Filename is derived from repo name.
`repo_name` - Sets name of the repo and used to construct path to password file

If `restic_use_password_file: true`, you need to set path to the password file in the `restic_repos[].password` to  `{{restic_password_dest}}/name_of_your_repo`
There must be a better way of handling this.

Cron template has if/else logic to set `RESTIC_PASSWORD_FILE` or `RESTIC_PASSWORD` variables, based on the boolean above.

In our case we use Ansible Tower and a playbook like this to provision restic clients:
```
- hosts: all
  become: true
  roles:
    - {role: ndench.environment, tags: "environment"}
    - {role: paulfantom.restic, tags: "restic-client"}
```
Hostvars:
```
---
restic_password: 'mykewlpassword'
restic_repo: 's3:http://minio:9000/my-repo'
repo_name: 'backupserver'
restic_repos:
- name: "{{ repo_name }}"
  url: "{{ restic_repo }}"
  password: "{{ restic_password_dest }}/{{ repo_name }}"
  remote_credentials:
    aws_secret_key_id: "{{ aws_key_id }}"
    aws_secret_access_key: "{{ aws_key }}"
  jobs:
    - command: 'restic backup /srv'
      at: '0 1  * * *'
  retention_time: '17 5 * * *'
  retention:
    last: 2
    hourly: 4
    daily: 10
    weekly: 9
    monthly: 3
    yearly: 10
```
Playbook run variables:
```
---
restic_version: '0.9.0'
restic_user: root
restic_group: "{{ restic_user }}"
restic_install_path: '/usr/local/bin'
restic_use_password_file: true
restic_password_dest: '/root/.restic'
restic_initialize_repos: true
repo_name: backupserver
aws_key_id: "our_minio_key_id"
aws_key: "our_minio_key"

environment_files:
  env:
    path: /etc/environment
    format: system
    owner: root
    group: root

environment_config:
  - key: RESTIC_PASSWORD_FILE
    value: "{{ restic_password_dest }}/{{ repo_name }}"
    files: [env]
  - key: RESTIC_REPOSITORY
    value: "{{ restic_repo }}"
    files: [env]
  - key: AWS_SECRET_ACCESS_KEY
    value: "{{ aws_key }}"
    files: [env]
  - key: AWS_ACCESS_KEY_ID
    value: "{{ aws_key_id }}"
    files: [env]
```
`ndench.environment` role sets all required variables to /etc/environment and configure.yml from restic role loads them before initialising the repo.

**Line 41 of configure.yml - is not working!**
In this case we do not need to pass any environment variables to init task at all (we use minio backend, and it requires key, id and url variables), but to make this role independent from other roles, as a minimum, we tried to set `RESTIC_PASSWORD` or `RESTIC_PASSWORD_FILE` variable in this task, but without luck. 
It should create: `RESTIC_PASSWORD: password` or `RESTIC_PASSWORD_FILE: /root/.restic/backupserver`

Hope you will have some input/ideas on how to handle this.
One option is to remove environment variable in init task and set dependency in meta file to `ndench.environment`, or try to implement the logic of setting the correct variable during init run, although it will limit us to use only local backend.


TODO:

- Environment vars creation need to iterate through array of repos and create all of them. Currently only 1 repo is allowed per host, which is enough for our case.
- We are setting some vars in hostvars and template vars to use them in both roles. Which is kind of confusing.