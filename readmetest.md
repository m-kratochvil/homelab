# Ansible for Converged Cloud network equipment
This is an internal SAP repository. Only members of the **CCloud-L4** and **ccloud-net-admins** teams are allowed to execute the code from the repository environment itself, others, such as MSP staff are welcome to contribute but code execution is allowed only through the [CND AWX](https://awx-prod.netinfra.c.eu-de-1.cloud.sap/#/login).

## **Repository rules and guidelines**
- **Branch protection, code reviews**
  - master branch is protected in this repository
  - code review is required to merge into the master branch
  - code owners are implemented
- **Playbook credentials:**
  - we use "per technical domain" vault files with “vault-id”
  - all passwords must be vault-secured
- **Playbook targets:**
  - It is mandatory to limit the targets in the playbooks to only the devices required 
  - this is enforced during PR code review
  - reviewer makes sure playbook does not contain `hosts: all` or similar dangerous setting that would target unintended devices
- **Use of variables (inventory file, host/group vars, separate var files, CLI, Netbox config-context...):**
  - use of shared variables is strongly recommended
  - until further notice, we let everyone use/call variables as they wish
  - config-context is disabled for the time being
  - keep variable definitions short
- **Use of branches:**
  - long-running development branches, such as `dev/firewall` can be deleted only by the branch owner
  - “patch” type of branches should be deleted right after merging PR from such branch to master
- **Files / Directories**
  - do not upload binary files to the repository. OS images, software packages etc. should be either kept locally or downloaded to the target device from cloud storage such as the CC Swift
  - for the purpose of external storage for this repository, we have the `repo` Swift container available. Contents of this storage replicate to all CC regions moments after uploaded.  
    - Container URL (for file management):  
      `https://dashboard.<region>.cloud.sap/ccadmin/master/object-storage/containers/repo/list`
    - Example URI to download a file from the `f5` directory, within a playbook task:  
      `https://repo.qa-de-1.cloud.sap/f5/f5-appsvcs-3.18.0-4.noarch.rpm`
- **Documentation:**
  - playbook are to be documented in the respective playbook directories
  - links to the playbook specific documentation to be added to the main readme.md (this one)
  - the main “readme.md” to be updated by everyone as needed, pending code review

### **Environment notes and recommendations**
The repository environment can be installed on various platforms and systems. The choice is up to you, the end user, but we do provide some general guidelines and recommendations below. Please note that we have no means to engage in detailed troubleshooting of platform related issues. Search engine is your friend.

If you are advanced user, you can simply clone the repository to your preferred environment, make sure you have the [dependencies](#4-dependencies) installed and you should be good to go.

For others, we provide recommendations and more detailed [guide](#environment-setup) below. 

### Tested environments
| Host platform | Guest platform | Notes / known issues, caveats |
| --- | --- | --- |
| Windows | VMware/Ubuntu 19.04 | Works without issues, using sshuttle as proxy |
| Windows | VMware/Ubuntu/Docker | Works without issues |
| Windows | VMware/Ubuntu 18.04, 19.10 | sshuttle frequently disconnects |
| Windows | WSL Ubuntu 18.X, 20.X | sshuttle not working, proxycommand used instead, some limitations reported,<br>e.g. some devices unreachable, some vendor modules not working |
| Windows       | Docker on WSL | Docker does not run on WSL as-is, need a Docker on Windows installed<br>See [this](https://medium.com/@callback.insanity/using-docker-with-windows-subsystem-for-linux-wsl-on-windows-10-d2deacad491f) guide. |
| MAC | MAC OS | sshuttle not working over the F5 Edge VPN |

### Environment setup
1. [Machine Setup](#1-machine-setup)
2. [Running on Docker](#2-running-on-docker)
3. [Repository and execution environment install](#3-repository-and-execution-environment-install)
4. [Dependencies](#4-dependencies)
   - [Python code](#python-code)
   - [Roles](#roles)
5. [Inventory](#5-inventory)
   - [Netbox](#netbox)
   - [YAML `vars/`](#yaml-vars)
6. [Specific playbook READMEs](#6-specific-playbook-readmes)
   - [Firewalling](#firewalling)
   - [Secrets](#secrets)
     - [**IMPORTANT!!!** RADIUS credentials:](#important-radius-credentials)
     - [HOWTO add, encrypt and edit `secret.yml`](#howto-add-encrypt-and-edit-secretyml)
7. [Using a local copy of the inventory to prevent long loading times](#7-using-a-local-copy-of-the-inventory-to-prevent-long-loading-times)

### 1. Machine Setup
We aim to run anisble on python3 as 2.7 is running out of support in 2020. Any *NIX system with the following components installed can run the repository code:
* git
* make
* python3.5 or higher
* python3-venv
* python3-setuptools
* virtualenv
* wget
* python3-dev
* python3-pip
* libffi-dev

On apt-based systems with recent repositories (such as ubuntu 18.04), these dependencies should be included in the standard sources and can be installed with:
```
sudo apt-get install git make python3 wget python3-venv python3-dev python3-pip python3-setuptools virtualenv libffi-dev
```

**Important:** If your systems default python3 interpreter (find with `python3 -V`) is anything below python 3.5, you must switch the systems default python3 interpreter or create the python virtual environment explicitly with the `python3.5` binary.

### 2. Running on Docker
Having ansible go over a jumphost everytime did not prove to be reliable. We therefore built a docker container that is able to run ansible.

After you clone the repository, start the container executing `./start-docker.sh`. It will automatically mount the ccloud-net directory into `/ccloud-net` of the docker container. The script asks you for the user to use for authentication against ssh endpoints. If you have the need to tunnel all traffic, through a jumphost, use `sshuttle-eu` which will use `jump01.cc.eu-de-1.cloud.sap` as tunnel host.

### 3. Repository and execution environment install
Clone this repo using ssh (if ssh key or ssh-agent forwarding present) or alternitvely https
```bash
git clone https://github.wdf.sap.corp/Infrastructure-Automation/ccloud-net.git
#or
git clone git@github.wdf.sap.corp:Infrastructure-Automation/ccloud-net.git
```

We will not modify the global python interpreter packages, instead we bootstrap a private ansible execution evironment. Do this with:
```bash
cd ccloud-net
make install
```
This will create a python virtual environment, install ansible and all python modules and external roles.

Ansible will not touch the global python interpreter, it will only reside in the virtual environment (venv). Thus this venv has to be activated before executing ansible. This can be achieved by executing
```
source venv/bin/activate
```

### 4. Dependencies
Ansible has dependencies towards python code and also ansible roles expressed in YAML.

  - #### Python code
    Dependencies to python code, e.g. ansible modules written in python, reside in the `plugins` folder. Ansible modules that require native python modules must register the dependency in the `requirements.txt` file in this repo's root directory. Python imports should include the minimum required code version, e.g.:
```
ansible>=2.7          <-- requires ansible 2.7 or later
pyIOSXR==0.6.44       <-- requires exactly this version
```

To update dependencies run
```
make update
```

  - #### Roles
    Roles can either be
    1. hard-copied in this repo or (residing in `roles/`),
    2. maintained elsewhere and then dependency managed via ansible-galaxy (residing in `roles.galaxy/`).

    For option 1), just nest the role in the `roles/` folder and start using it. Option 2) requires you to register the role and from where to install it in the `requirements.yaml`. We recommend to use some sort of versioning, either by branches or by git tags. See the examples below:

    ```yaml
    - src: git@github.wdf.sap.corp:NetInfra/role_nested_config.git
      scm: git
      version: master
      name: nested_config

    - src: git@github.wdf.sap.corp:NetInfra/role_extcommunity_list.git
      scm: git
      tag: 1.0
      name: extcommunity_list
    ```

    Galaxy managed roles can then be installed using `ansible-galaxy install -r requirements.yaml`

    Roles that are tightly coupled to CCloud equipment should reside in this repo, roles that are universal and can be used for a broader set of devices or are likely to be imported in a variety of playbooks should reside in a seperate repo. There is already a decent selection of ansible roles in the [NetInfra GitHub](https://github.wdf.sap.corp/NetInfra?utf8=✓&q=role_&type=&language=).

### 5. Inventory
  - ####  Netbox
    Basic host information will be sourced from netbox. Connection and query parameters are set in `inventories/000_netbox.yaml`. Adjust the query filters to include your device role, but keep them lean as the inventory build-up can take a bit. *Make sure to filter the hosts you are applying a playbook to in the playbook itself.*
    You can run a `ansible-inventory --list` to have a look on the inventory contents and structure.

  - #### YAML `vars/`
    Netbox config context delivers the possibility to apply variables on multiple criteria. However, these variables are world-readable, with absolutely no auditing and very few accounting options. Hence, a second option to include variables are yaml files. These files need to be explicitly imported during the playbook run. You can either import them generically (for all hosts this playbook applies to) or specific on a hosts attribute. For instance:

    ```yaml
    - hosts: Neutron_Router
      gather_facts: yes
      vars_files:
        - ../../vars/network/network_auth.yaml
        - ../../vars/network/logging.yaml
        - ../../vars/network/neutron_ntp.yaml
        - ../../vars/network/neutron.yaml
        - ../../vars/network/snmp.yaml
        - ../../vars/network/prefix_lists/deny_all.yaml
        - ../../vars/network/access_lists/acl_cisco_scanner.yaml
        - ../../vars/network/access_lists/acl_snmp_vty.yaml
    ```
    is the head for a playbook running for all routers that have the role 'Neutron_Router' assigned, all vars are statically imported no matter in which region that router is in.

    Dynamic imports can be done in the "tasks" section like this:
    ```yaml
    tasks:
    - name: Include site specific variables
      include_vars: "../../vars/network/neutron_{{ site }}.yaml"
    ```
    This will replace `{{ site }}` with the value of the `site` attribute of the router it currently processes. E.g. it will call the file `vars/network/neutron_qa_de_2a.yaml` for a router attributed accordingly.

    **All variable paths are relative to the playbooks location**

### 6. Specific playbook READMEs

  - ### Firewalling

    - General

      https://github.wdf.sap.corp/Infrastructure-Automation/ccloud-net/blob/dev/firewall/playbooks/firewalling/README.md

    - Firepower

      https://github.wdf.sap.corp/Infrastructure-Automation/ccloud-net/blob/master/playbooks/firewalling/firepower/README.md

    - ASA multi context

      https://github.wdf.sap.corp/Infrastructure-Automation/ccloud-net/blob/master/playbooks/firewalling/multi-context/README.md

    - ASA single context

      https://github.wdf.sap.corp/Infrastructure-Automation/ccloud-net/blob/master/playbooks/firewalling/single-context/README.md


  - ### Secrets

    - #### **IMPORTANT!!!** RADIUS credentials:

      RADIUS credentials Key **must** be added to`/vars/secret.yml`:

    - #### HOWTO add, encrypt and edit `secret.yml`

      1. Add file (with the following relative path if you are in cc-cloud-net directory)
          ```
          touch vars/secret.yml
          ```

      2. Encrypt file  
          ```
          ansible-vault encrypt vars/secret.yml
          ```

          `vars/secret.yml` is now encrypted

      3.  Edit file:

           ```
           ansible-vault edit vars/secret.yml
           ```

      4. Format:

          ```yaml
          ---
          radius_username: [your-username]
          radius_password: [your-radiuspassword]
          ```
### 7. Using a local copy of the inventory to prevent long loading times

  1. Generate a local copy of the inventory
      ```
      ansible-inventory -y --list > inventories/000_local_copy.yaml
      ```
  2. Prevent ansible from loadinng the cache or other inventories by editing `ansible.cfg`. Make sure the following lines look equal:
      ```ini
      [default]
      ...
      inventory=./inventories/000_local_copy.yaml
      ...
      [inventory]
      enable_plugins=yaml
      cache=false
      ```
  3. Do not commit your locally changed `ansible.cfg`!
