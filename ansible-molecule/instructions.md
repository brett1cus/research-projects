# Molecule - an Ansible testing framework

## Overview
Molecule offers a broad range of functionality that greatly reduces the complexity/headache of setting up/tearing down 
testing/development environments in relation to Ansible roles. In this exercise we will run through the steps needed
to create a simple proof of concept that goes over the key elements of Molecule.

## Prequisites
Any work done should be performed on a test server of some description, most likely a throwaway VM running some flavour of
Linux. For the purpose of this exercise, we will be using a Centos 7.7 server. 

You should have internet access, have the ability to pull down packages using pip and Docker images from both
[dockerhub](https://hub.docker.com/) and [quay.io](https://quay.io). This guide assumes that Docker is already installed on the test server.

## Proof of Concept instructions
By following the steps outlined below, you should be able to complete an end to end test of a simple Ansible role. 

  1. To get everything installed, follow [these](https://molecule.readthedocs.io/en/stable/installation.html#install) instructions from Molecules' official readthedocs site.

  2. We create a virtual environment that will contain all of Molecules' dependencies and then activate it. If for some reason you don't have virtualenv installed on your Centos box, you can run `sudo yum install python-virtualenv`. For this exercise we will use /opt for the virtualenv location and "molecule" for its name, but you can use whatever you prefer:

```
$ cd /opt
$ virtualenv molecule 
$ . molecule/bin/activate
```
	

  3. We can now actually install the Molecule dependencies using pip. Notice that the second pip install actually installs the additional Docker libraries into the virtualenv; for now, we will be using Docker to test our Ansible code:

```
(molecule) pip install molecule
(molecule) pip install 'molecule[docker]'
```

  4. For this stage we will need an actual Ansible role. For convenience, we have one already prepared. Assuming you have cloned this repository and you are in the `ansible-molecule` directory (which is probably a good idea), if you perform a `tree` on the `roles/simple_ansible_role` directory you should see the following:

```
(molecule) [root@testserver ansible-molecule]# tree roles/simple_ansible_role
simple_ansible_role
├── files
│   └── witcher_lines.txt
└── tasks
    └── main.yml
```

So, a very straightforward Ansible role that copies a text file to /tmp.

  5. Now we can actually start using Molecule! There are two methods of invoking Molecule, depending on your situation. If you need to create a brand-new role, you would let Molecule take care of the directory structure creation (it uses Ansible Galaxy under the hood), so something like `molecule init role -r my-new-role`. However, as we want to retrofit Molecule into an already existing Ansible role, we would do the following:

```
(molecule) cd roles/simple_ansible_role
(molecule) [root@testserver simple_ansible_role]# molecule init scenario -r simple_ansible_role
--> Initializing new scenario default...
Initialized scenario in /opt/research-projects/ansible-molecule/roles/simple_ansible_role/molecule/default successfully.
```

We can repeat the `tree` command as in step 4 to see some interesting additions:

```
(molecule) [root@testserver ansible-molecule]# tree roles/simple_ansible_role
roles/simple_ansible_role
├── files
│   └── witcher_lines.txt
├── molecule
│   └── default
│       ├── Dockerfile.j2
│       ├── INSTALL.rst
│       ├── molecule.yml
│       ├── playbook.yml
│       └── tests
│           ├── test_default.py
│           └── test_default.pyc
└── tasks
    └── main.yml
```

By initialising Molecule, we now have a suite of tests available, albeit with a bit of tweaking. The following is from the official readthedocs:

  Since Docker is the default Driver, we find a Dockerfile.j2 Jinja2 template file in place. Molecule will use this file to build a docker image to test your role against.

  INSTALL.rst contains instructions on what additional software or setup steps you will need to take in order to allow Molecule to successfully interface with the driver.

  molecule.yml is the central configuration entrypoint for Molecule. With this file, you can configure each tool that Molecule will employ when testing your role.

  playbook.yml is the playbook file that contains the call site for your role. Molecule will invoke this playbook with ansible-playbook and run it against an instance created by the driver.

  tests is the tests directory created because Molecule uses TestInfra as the default Verifier. This allows you to write specific tests against the state of the container after your role has finished executing. Other verifier tools are available.

  6. Now we can instantiate a Docker container to run the Ansible role against. However, before we do, we will need to add a line into the Dockerfile.j2 because the role sets the owner of the copied file to someone that doesn't exist in the container yet! Add the following line into Dockerfile.j2:

```
RUN useradd geralt -s /bin/bash -u 55555
```

  7. Run `molecule create` and you should hopefully see the following:

```
(molecule) [root@testserver simple_ansible_role]# molecule create
--> Validating schema /opt/research-projects/ansible-molecule/roles/simple_ansible_role/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix
    
└── default
    ├── dependency
    ├── create
    └── prepare
    
--> Scenario: 'default'
--> Action: 'dependency'
Skipping, missing the requirements file.
--> Scenario: 'default'
--> Action: 'create'
--> Sanity checks: 'docker'
    
    PLAY [Create] ******************************************************************
    
    TASK [Log into a Docker registry] **********************************************
    skipping: [localhost] => (item=None) 
    
    TASK [Create Dockerfiles from image names] *************************************
    changed: [localhost] => (item=None)
    changed: [localhost]
    
    TASK [Determine which docker image info module to use] *************************
    ok: [localhost]
    
    TASK [Discover local Docker images] ********************************************
    ok: [localhost] => (item=None)
    ok: [localhost]
    
    TASK [Build an Ansible compatible image (new)] *********************************
    changed: [localhost] => (item=molecule_local/centos:7)
    
    TASK [Build an Ansible compatible image (old)] *********************************
    skipping: [localhost] => (item=molecule_local/centos:7) 
    
    TASK [Create docker network(s)] ************************************************
    
    TASK [Determine the CMD directives] ********************************************
    ok: [localhost] => (item=None)
    ok: [localhost]
    
    TASK [Create molecule instance(s)] *********************************************
    changed: [localhost] => (item=instance)
    
    TASK [Wait for instance(s) creation to complete] *******************************
    changed: [localhost] => (item=None)
    changed: [localhost]
    
    PLAY RECAP *********************************************************************
    localhost                  : ok=7    changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
    
--> Scenario: 'default'
--> Action: 'prepare'
Skipping, prepare playbook not configured.
```

But how does Molecule know what Docker image I want? Well, there's a really useful file called `molecule.yml` that allows you to configure what platforms, verifiers, drivers and linters you want to use:

```
---
dependency:
  name: galaxy
driver:
  name: docker
lint:
  name: yamllint
platforms:
  - name: instance
    image: centos:7
provisioner:
  name: ansible
  lint:
    name: ansible-lint
verifier:
  name: testinfra
  lint:
    name: flake8
```

  8. Now the Centos:7 image has been pulled in, let's apply the Ansible role to a new container instantiated from that newly created image via `molecule converge` (Note: Molecule has a neat feature that will create a playbook that calls the role, so you don't have to worry about creating a testing play separately):

```
(molecule) [root@testserver simple_ansible_role]# molecule converge
--> Validating schema /opt/research-projects/ansible-molecule/roles/simple_ansible_role/molecule/default/molecule.yml.
Validation completed successfully.
--> Test matrix
    
└── default
    ├── dependency
    ├── create
    ├── prepare
    └── converge
    
--> Scenario: 'default'
--> Action: 'dependency'
Skipping, missing the requirements file.
--> Scenario: 'default'
--> Action: 'create'
Skipping, instances already created.
--> Scenario: 'default'
--> Action: 'prepare'
Skipping, prepare playbook not configured.
--> Scenario: 'default'
--> Action: 'converge'
    
    PLAY [Converge] ****************************************************************
    
    TASK [Gathering Facts] *********************************************************
    ok: [instance]
    
    TASK [simple_ansible_role : Copy a file with some cool dialogue in it to /tmp] ***
    ok: [instance]
    
    PLAY RECAP *********************************************************************
    instance                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

And that's it, we have lift off! We can check that the file has indeed been copied within the container by leveraging a nice wrapper function courtesy of molecule that allows us to go into the container itself:

```
(molecule) [root@testserver simple_ansible_role]# molecule login
--> Validating schema /opt/research-projects/ansible-molecule/roles/simple_ansible_role/molecule/default/molecule.yml.
Validation completed successfully.
[root@instance /]# cat /tmp/witcher_lines.txt 
Geralt: "Hmmmmmmmmmmmm...Molecule looks pretty good..."

[root@instance /]# ll /tmp/
total 8
-rwx------. 1 root   root   836 Oct  1 01:16 ks-script-56tHfe
-rwxrwxr-x. 1 geralt geralt  56 Jan 17 12:41 witcher_lines.txt
-rw-------. 1 root   root     0 Oct  1 01:15 yum.log

[root@instance /]# id geralt
uid=55555(geralt) gid=55555(geralt) groups=55555(geralt)

```

That looks like what we hoped it would be. We can improve on this by adding some tests and using TestInfra's capabilities.
