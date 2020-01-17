# Molecule - an Ansible testing framework

## Overview
Molecule offers a broad range of functionality that greatly reduces the complexity/headache of setting up/tearing down 
testing/development environments in relation to Ansible roles. In this exercise we will run through the steps needed
to create a simple proof of concept that goes over the key elements of Molecule.

## Prequisites
Any work done should be performed on a test server of some description, most likely a throwaway VM running some flavour of
Linux. For the purpose of this exercise, we will be using a Centos 7.7 server. 

You should have internet access, have the ability to pull down packages using pip and Docker images from both
[dockerhub](https://hub.docker.com/) and [quay.io](https://quay.io).

## Proof of Concept instructions
By following the steps outlined below, you should be able to complete an end to end test of a simple Ansible role. 

  1. To get everything installed, follow [these](https://molecule.readthedocs.io/en/stable/installation.html#install) instructions from Molecules' official readthedocs site.

  2. We create a virtual environment that will contain all of Molecules' dependencies and then activate it. If for some reason you don't have virtualenv installed on your Centos box, you can run `sudo yum install python-virtualenv`. For this exercise we will use /opt for the virtualenv location and "molecule" for its name, but you can use whatever you prefer:

```
$ cd /opt
$ virtualenv molecule 
$ . molecule/bin/activate
```
	

  3. We can now actually install the Molecule dependencies using pip. Notice that the second pip install actually installs Docker into the virtualenv; for now, we will be using Docker to test our Ansible code:

```
(molecule) pip install molecule
(molecule) pip install 'molecule[docker]'
```

  4. For this stage we will need an actual Ansible role. For convenience, we have one already prepared. Assuming you have cloned this repository and you are in its root directory, if you list the contents of `simple_ansible_role` you should see the following:

```
(molecule) 
