# Ansible test project

The object of this repo is offer an example for **full provisioning a simple web app with Ansible**, including server's 
software installation/update, database creation and app code deployment with [`Ansistrano`](https://github.com/ansistrano)
 
If you want to test this example easily:

1. Install Ansible and Ansible roles as described in first section.
2. Add your test servers' IPs at inventories config files. I recommend 
[DigitalOcean](https://www.digitalocean.com/?refcode=d75130f445da): you can run & destroy cloud servers in seconds. 
Really good service & prices, starting at $5/month, so you can run your tests without almost any cost. If you create your 
account from the previous link, both of us will receive a $10 bonus, so ;-)
3. Execute some of the provision / deploy examples available in section 4.

Enjoy!

## 1. Install Ansible

On the provisioner/deployer machine (or your local computer):

- On MacOS X, install XCode. You should have Command Line Tools installed. You can do it via terminal: `xcode-select --install`
- `sudo easy_install pip`
- `sudo pip install ansible --quiet`
- `sudo pip install cryptography` (will make the encrypt/decrypt operations faster for ansible-vault)

### Install additional Ansible roles being used in this project

This project uses the following Ansible roles, available publicly at Ansible Galaxy directory:

- geerlingguy.apache
- geerlingguy.php
- geerlingguy.composer
- geerlingguy.memcached
- geerlingguy.mysql
- carlosbuenosvinos.ansistrano-deploy
- carlosbuenosvinos.ansistrano-rollback

You should install all of them using `sudo ansible-galaxy install {{ role_name }}`.

You can install several roles at once: 
`sudo ansible-galaxy install {{ role_name_1 }} {{ role_name_2 }} {{ role_name_3 }}[...]`

## 2. Key concepts

### 2.1 Inventories

Definition files that include hosts where the tasks should be executed. They can be organized in groups and include
its own specific variables that can be referenced from other parts (to build paths, reference files, launch conditional 
tasks, write files based on templates, etc.).

In our example, we've defined two main inventories, `production` and `staging`. Both inventories contain two main groups 
of hosts, `web` and `db`. These two groups are included into a generic `app` group, so we can assign some variables that 
will be used internally, when those hosts are used, depending on the environment we are using.

### 2.2 Roles

Roles offer a way to combine tasks, handlers, variables, templates... that share a specific responsibility in the server.

Some role examples could be: 

- `apache`, `mysql`, `memcache`...: for provision the software, offer some templates to 
configure the service, define the dependencies (if it depends on other roles)
- `common_tasks`: example included in this 
project that take some common tasks to be done in a clean system (install and update software repos, install common 
packages, configure and start time synchronization, etc.)
- `deploy`, `rollback`: these roles come from [`Ansistrano`](https://github.com/ansistrano), a port of 
`Capistrano`, the well known deploy tool, using `Ansible`. They include all common tasks involved in an app
 deploy process, including multiple deploy strategies, keeping previous releases, rollback
 changes, etc.

### 2.3 Playbooks

Playbooks allow us to combine roles & hosts, telling where do we want to execute what. In our example, we have 3 
"provision" playbooks that install & configure needed software for a basic web app on a simple servers infrastructure: 

- `provision_web` tell that hosts marked as `web` should execute roles `common_tasks`, `apache`, `php`, `composer`, 
`memcached`.
- `provision_db` tell that hosts marked as `db` should execute roles `common_tasks`, `mysql` 
- `provision_all` include two previous playbooks, so web can launch all tasks together.
- `app_deploy` & `app_rollback`: include the `Ansistrano` roles to deploy a basic web app to given hosts. 

## 3. Project structure. Files & folders.

### 3.1 Common 

- `ansible.cfg`: This store main configuration options for Ansible and overwrite the default values for every key 
included. The only two parameters we're overwritting in this test are `host_key_checking`, set to false only for testing
 reasons (avoid host check while testing provisioning on cloud servers being rebuilt every while). 
- `vpass`: Ansible offers an easy way to encrypt/decrypt files to keep sensible information like passwords. For instance
 for those provision roles that include database creation with passwords, or services that require authentication information.
  The encription tasks are managed by [`ansible-vault`](http://docs.ansible.com/ansible/playbooks_vault.html). It can 
  read a master password from a static file (it's the case), so you'll not be asked for it in every execution. The 
  master password in this file will be stored in plain text, so **this file should not be included in the repository**! 
  This project include `./vpass` in main `.gitignore`. If you want to test this project, you can add the file by 
  yourself: the master password of this test is '123456', so this is the content you should add. ;-)
- `inventories`: contains hosts & groups files. You can create different files for every environment (production,
staging, development) 
- `group_vars/{{ group|host }}/*.yml`: define variables that will apply when tasks are being applied to specified host or group.
For instance, we create here the variables that will be when installing `apache` role to `web` group.

### 3.2 Provision

- `roles/{{ role }}/files`: source files that could be used by role's tasks. For instance config files, keys... 
- `roles/{{ role }}/templates`: templates based on [`Jinja2`](http://jinja.pocoo.org/docs/dev/) for build files used by
the role.
- `roles/{{ role }}/tasks`: main tasks 
- `roles/{{ role }}/handlers`: tasks that should be launched every time tasks are executed. For instance: on role `apache`,
every time we execute the main provision tasks, we should restart `httpd` service.
- `roles/{{ role }}/vars`: configuration / templating variables that affect this role. 
- `roles/{{ role }}/defaults`: default values for role variables.
- `roles/{{ role }}/meta`: role dependencies.

### 3.3 Deploy
- `deploy_tasks/{{ task_name }}.yml` & `deploy_tasks/{{ app }}/{{ task_name }}.yml`: Include those tasks that will be 
executed during the deploy process. For instance: launch `composer install` after deploying code to new release, and 
before changing symbolic for `current` release folder.
- `deploy_vars/{{ app }}.yml`: define variables that will be used during the deploy. For instance: The URL of the app's
git repo, the tasks that will be launched in the app deploy and when.


## 4. Some examples & use cases

- **Provision web**<br/>
  `$ ansible-playbook -i inventories/production provision_web.yml`

- **Provision all**<br/>
  `$ ansible-playbook -i inventories/production provision_all.yml`

- **Deploy app 'myapp' to production environment**<br/>
  `$ ansible-playbook -i inventories/production app_deploy.yml -e "app=myapp"`

- **View contents of encrypted file**<br/>
  `$ ansible-vault view group_vars/all/passwords.yml`

## 5. Good practices

### Authentication

- You need to log into every one of the servers included in the hosts files. You should do it by using ssh keys,
 so you don't have to use or store ssh passwords.
- Some of the tasks will involve authentication with third parties from the remote servers. For instance, when you are 
 deploying an application using the `git` strategy. Unless you have added every server ssh keys to your GIT repos (not
  recommended), you should enable ssh agent forwarding, so your local ssh keys (or those from your deploying server) will 
  be used when authenticating against the third party services. Your remote servers will act only as pass-through.
- Remember to encrypt those files with sensible information using [`ansible-vault`](http://docs.ansible.com/ansible/playbooks_vault.html). 
 You can assign passwords, api_keys, etc. to variables and then use those variables from your configuration files. You'll find
 an example in this project with `group_vars/all/passwords.yml` (encrypted passwords) and `group_vars/db/mysql.yml` (use password 
 variables setted in the encrypted file).
 
Corrections, contributions and PR are more than welcome.
