# How to write an Ansible Playbook

Here's some notes about the Ansible tool.

## Directory structure

The tool should not provide only the playbook itself but a full Ansible directory with the following structure according to the standard directory structure suggested by the Ansible docs.

```
production                # inventory file for production servers
staging                   # inventory file for staging environment

group_vars/
   group1.yml             # here we assign variables to particular groups
   group2.yml
host_vars/
   hostname1.yml          # here we assign variables to particular systems
   hostname2.yml

site.yml                  # master playbook
webservers.yml            # playbook for webserver tier
dbservers.yml             # playbook for dbserver tier

roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies
        library/          # roles can also include custom modules
        module_utils/     # roles can also include custom module_utils
        lookup_plugins/   # or other types of plugins, like lookup in this case

    webtier/              # same kind of structure as "common" was above, done for the webtier role
    monitoring/           # ""
    fooapp/               # ""
```

### Inventory files

In the inventory files you should list the target machines on which your playbooks should operate. In the example above we have two different inventories, one containing the machines used in production and the other for the machines used in for staging (these two files should be filled in by the user, or I should include a particular field for specifying the machines in the description file of my tool, we'll see). In the inventories you can define variables for both groups of hosts and single hosts, however this practice is discuraged, it's better to define these variables in separated files. The only types of variables one could decide to put in the inventory directly are those related on how to connect to the hosts.

Here's an example of inventory file.

```
# file: production

[atlanta_webservers]
www-atl-1.example.com
www-atl-2.example.com

[boston_webservers]
www-bos-1.example.com
www-bos-2.example.com

[atlanta_dbservers]
db-atl-1.example.com
db-atl-2.example.com

[boston_dbservers]
db-bos-1.example.com

# webservers in all geos
[webservers:children]
atlanta_webservers
boston_webservers

# dbservers in all geos
[dbservers:children]
atlanta_dbservers
boston_dbservers

# everything in the atlanta geo
[atlanta:children]
atlanta_webservers
atlanta_dbservers

# everything in the boston geo
[boston:children]
boston_webservers
boston_dbservers
```

### Group/Host vars files

In the `group_vars` and `host_vars` directories contains files where you can define variables referring to group of hosts or to single hosts you defined in in the inventories.

For example following the previous example, we could define in the `group_vars` folder a file like the following.

```
---
# file: group_vars/atlanta.yml
ntp: ntp-atlanta.example.com
backup: backup-atlanta.example.com
```

We have to keep in mind that while the inventories can be written in the INI format, variables files must be written in YAML hence they must start with the 3 dashes.

Moreover there are two implicit gropus defined by default for which we can even define an associated variable file, `all` and `ungrouped`, the first groupes all the hosts defined, the second groups all the hosts which don't have a group.

### Playbooks

In the above example we defined three top level playbooks. Usually we have a "main" playbook, in this case site.yml, which includes all the other top level playbooks. These top level playbooks are divided by role and the idea is to create a hierarchy for which by running the site.yml playbook you run all the playbooks defined for the whole infrastructure instead by running one of the other top level playbooks you act on a subset of the machines of the infastructure. A short example of what `site.yml` could you would be just import the other two top level playbooks.

```
---
# file: site.yml
- import_playbook: webservers.yml
- import_playbook: dbservers.yml
```

while these playbooks could for example use the playbooks defined in the roles.

```
---
# file: webservers.yml
- hosts: webservers
  roles:
    - common
    - webtier
```

by organizing the playbooks hierarchycally in this way we can explicitely act on a subset of the machines grouped by some criteria in a more explicit way than when using the `--limit` argument in the `ansible-playbook` command.

```
ansible-playbook site.yml --limit webservers
```
would be equal to
```
ansible-playbook webservers.yml
```

### Roles

Roles are bundles that include playbooks, variables, handlers, templates and anything a playbook would need to automate a particular process. In detail the above folder structure presents:

- `tasks`: in this folder we place the playbooks for the role, there must be at least one `main.yml` main playbook that can include other minor playbooks in this same folder.
- `handlers`: here we place our handlers, an handler is a particular task that is been triggered by another task that includes a `notify` targeting that handler. The `notify` plugin has effect only when the task that includes it executes changing the state on the target machine. 
- `templates`: here we place our jinja2 templates, templates are files (usually configuration files) that must be filled at task running time with values coming from defaults, variables or facts gathered from the machine (default facts or custom ones). An example of template could be a configuration file for a web server like apache2 that will contain different informations depending on the particular machine it will be copied on.
- `files`: contains miscellanious files like files that has to be copied on the target machines or script that must be therein executed
- `vars`: contains YAML files describing variables to be used in the playbooks of this role
- `defaults`: are lower priority variables with respect to the variables.
- `meta`: contains the dependencies of the role (other playbooks, other roles, etc..)

The other folders won't be used by my tool probably.

An example of a role playbook.

```
---
# file: roles/common/tasks/main.yml

- name: be sure ntp is installed
  yum:
    name: ntp
    state: present
  tags: ntp

- name: be sure ntp is configured
  template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
  notify:
    - restart ntpd
  tags: ntp

- name: be sure ntpd is running and enabled
  service:
    name: ntpd
    state: started
    enabled: yes
  tags: ntp
```

here we have the usage of the `notify` plugin that targets an handler called `restart ntpd` that could look like this.

```
---
# file: roles/common/handlers/main.yml
- name: restart ntpd
  service:
    name: ntpd
    state: restarted
```

## Notes

Since I don't know in advance where the artefact will be depoyed I cannot produce inventories, at most I can produce the top level playbooks including a role I create (or I could even just create the role).

## Operating System and Distribution Variance

If our playbooks will contain tasks that must be executed only on certain types of OS than we can create dynamically groups of hosts using the automatic facts gathering plus the `group_by` module. For example:

```
---

 - name: talk to all hosts just so we can learn about them
   hosts: all
   tasks:
     - name: Classify hosts depending on their OS distribution
       group_by:
         key: os_{{ ansible_facts['distribution'] }}

 # now just on the CentOS hosts...

 - hosts: os_CentOS
   gather_facts: False
   tasks:
     - # tasks that only happen on CentOS go here
```

We can even define group variables in advance for these dynamically generated groups.

```
---
# file: group_vars/all
asdf: 10

---
# file: group_vars/os_CentOS
asdf: 42
```

And if we only need particular variables for a particular OS we can define the group variables as showed above and then include them in the playbook like this (here we just print the variable):

```
- hosts: all
  tasks:
    - name: Set OS distribution dependant variables
      include_vars: "os_{{ ansible_facts['distribution'] }}.yml"
    - debug:
        var: asdf
```


