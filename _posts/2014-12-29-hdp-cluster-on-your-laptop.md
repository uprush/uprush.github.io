---
layout: post
title: "HDP Cluster on Your Laptop"
description: "Building a 4 nodes HDP (Hortonworks Data Platform) cluster on your laptop with Apache Ambari and Vagrant is pretty simple."
category: "hdp"
tags: [hadoop, hdp, vagrant]
---
{% include JB/setup %}

Building a 4 nodes HDP (Hortonworks Data Platform) cluster on your laptop with Apache Ambari and Vagrant is pretty simple. This is useful for developing and testing Hadoop applications.

## System requirements
- More than 8GB RAM avaiable on the laptop for HDP vitual machines

## Install Vagrant
- Download and install [Vagrant](https://www.vagrantup.com).
- We will also need to install [Oracle VirtualBox](https://www.virtualbox.org) as the Vagrant Provider.

## Vagrantfile
We are ready to define VMs (virtual machines) to run HDP cluster on Vagrant. VMs are defined in the `Vagrantfile`.

Create a working directory.

    $ mkdir hdp22
    $ cd hdp22

Generate an example Vagrantfile. Following command will create the Vagrantfile in current directory.

    $ vagrant init

Common VM configuration.

    $ vi Vagrantfile

We configure Vagrant to use CentOS 6.6 as the base box.

    config.vm.box = "chef/centos-6.6"

We also make some basic provisioning to meet the system requirement to run HDP. Here we use Shell Provisioner.

    $script = <<SCRIPT
    sudo yum -y install ntp
    sudo chkconfig ntpd on
    sudo chkconfig iptables off
    sudo /etc/init.d/iptables stop
    sudo setenforce 0
    sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
    sudo sh -c 'echo "* soft nofile 10000" >> /etc/security/limits.conf'
    sudo sh -c 'echo "* hard nofile 10000" >> /etc/security/limits.conf'
    sudo sh -c 'echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag'
    sudo sh -c 'echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled'
    SCRIPT

    config.vm.provision "shell", inline: $script

We are now ready to define VMs. We will define 4 VMs, 1 Ambari server, 1 Hadoop master and 2 slaves.

    # Ambari1
    config.vm.define :ambari1 do |a1|
      a1.vm.hostname = "ambari1.mycluster"
      a1.vm.network :private_network, ip: "192.168.0.11"
      a1.vm.provider :virtualbox do |vb|
        vb.memory = "2048"
      end

      a1.vm.network "forwarded_port", guest: 8080, host: 8080
      a1.vm.network "forwarded_port", guest: 80, host: 80
    end

    # Master1
    config.vm.define :master1 do |m1|
      m1.vm.hostname = "master1.mycluster"
      m1.vm.network :private_network, ip: "192.168.0.12"
      m1.vm.provider :virtualbox do |vb|
        vb.memory = "4096"
      end
    end

    # Slave1
    config.vm.define :slave1 do |s1|
      s1.vm.hostname = "slave1.mycluster"
      s1.vm.network :private_network, ip: "192.168.0.21"
      s1.vm.provider :virtualbox do |vb|
        vb.memory = "2048"
      end
    end

    # Slave2
    config.vm.define :slave2 do |s2|
      s2.vm.hostname = "slave2.mycluster"
      s2.vm.network :private_network, ip: "192.168.0.22"
      s2.vm.provider :virtualbox do |vb|
        vb.memory = "2048"
      end
    end

A complete [Vagrantfile example](https://gist.github.com/uprush/a34c0e45c735f7979bea) is available for reference.

## Start VMs

We can start Ambari server VM from Vagrant now. On startup, Vagrant will automatically run the provision defined in Vagrantfile by Shell Provisioner.

    $ vagrant up ambari1

SSH in to Ambari server VM.

    $ vagrant ssh ambari1

We then using similar procedure to start the master and slave VMs.

## Install Ambari Server

SSH in to Ambari VM, install and set up Ambari Server. All the belows should be run as `root`.

    # Install
    wget -nv http://public-repo-1.hortonworks.com/ambari/centos6/1.x/updates/1.7.0/ambari.repo -O /etc/yum.repos.d/ambari.repo
    yum -y install ambari-server

    # Setup. There are several options to configure during setup.
    ambari-server setup

    # Start Ambari Server
    ambari-server start

## Set up VM

Before deploying HDP cluster by Ambari, we have some configuration to make.

FQDN setting. Add FQDN to each the `/etc/hosts` file on each VM.

    192.168.0.11 ambari1.mycluster ambari1
    192.168.0.12 master1.mycluster master1
    192.168.0.21 slave1.mycluster slave1
    192.168.0.22 slave2.mycluster slave2

Set up password-less SSH from Ambari server to all nodes.

On Ambari server:

    $ ssh-keygen
    $ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

On other nodes, copy content of `~/.ssh/id_rsa.pub` from Ambari server to `~/.ssh/authorized_keys`:

    $ cat >> ~/.ssh/authorized_keys


## Deploy HDP Cluster

We are ready to deploy a HDP cluster from Ambari Web UI. Because the UI is really simple, I would omit the screenshots here.

- Access [http://localhost:8080/](http://localhost:8080/) from your laptop. The username and password is admin and admin respectively.

- Give a cluster name.

- Select HDP 2.2

- Input hostname of the VMs (one per line) and the SSH private key of Ambari server. SSH user should be `vagrant`.

- Accept the default options for rest of the wizard.

- There are several credetials to configure.

- Complete the wizard. It takes about 30m to finish up.


We have our 4 nodes HDP cluster on a laptop.

![Ambari Dashboard](http://d2ovxh5wj5dcbc.cloudfront.net/images/ambari-dashboard-small.png)

Enjoy hadoop!

## What's next?
There are couple of things to do:

- Configure port forwording of the VMs to view web UI from browser of the laptop.
- Run the [Tutorials](http://hortonworks.com/tutorials/) on your HDP cluster.

**References**:

- [How to build a Hadoop VM with Ambari and Vagrant](http://hortonworks.com/blog/building-hadoop-vm-quickly-ambari-vagrant/)