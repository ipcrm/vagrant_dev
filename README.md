`vagrant_dev`
===============

1. [Overview](#overview)
2. [Setup](#setup)
    * [Prerequisites](#prerequisites)
    * [Defaults Configuration](#defaults-configuration)
    * [Platform Configuration](#platform-configuration)
    * [Role Configuration](#role-configuration)
3. [Puppet](#puppet)

## Overview

Basic vagrant repo that allows you to define one or more local VMs for development purposes.


## Setup
Start by cloning this repo to your machine.

### Prerequisites
The following vagrant plugins are needed:

```
vagrant-hosts (2.8.0+)
vagrant-norequiretty
```

### Defaults Configuration
To start, you're going to need to fill in your details in defaults.yml; the values in question:

```yaml
defaults:
  ssh_username:       <default username to use when ssh'ing to your instances (you can customize per image later)>
  communicator:       <default communicator.  Recommend setting this to ssh and overriding to winrm for windows(overrides are set elsewhere)>
  ssh_disabled:       <default status for ssh.  Recommend setting to false and overriding for windows to true (overrides are set elsewhere)>
  winrm_username:     <WinRM username to use with windows machines>
  winrm_password:     <WinRM password to use with windows machines>
  primary: false      <Default setting for if this should be the primary machine in a multi-master env.  This must be set to false, can override elsewhere>
  autoup: true        <Default setting for if the machine should be booted when an unqualified vagrant up is executed>
  cpus: 1             <Default CPU count>
  mem: 512            <Default memory in mb>
instance_defaults:
  box: 'dummy'        <The default box for vagrant.  Leave this set to dummy if you want to set for each instance you define - if they are all the same you can set here>
  sync_method: none   <The default sync method for vagrant.  Leave this to none by default and customize per image. Alt you can set this to 'rsync' if all your machines are going to be linux.>
  tcp_ports:          <A hash of ports to be forwarded for tcp>
  udp_ports:          <A hash of ports to be forwarded for udp>
  shared_folders:     <A hash of folders to be shared between the host and guest>
```

### Platform Configuration
A platform in terms of vagrant_dev represents the configuration settings required to stand up a particular 'configuration'.  For example, if you want to build out 3 RHEL machines you could create a platform with all of the required settings so you don't need to define for each instance.

Example platform configuration:

```yaml
---
platforms:
  centos7:
    sync_method: rsync
    box: 'puppetlabs/centos-7.2-64-nocm'
    shared_folders:
      '/Users/ipcrm/Downloads': '/tmp/downloads'
    tcp_ports:
      '80': '10000'
    roles:
      - posix_agent_standalone

  server2012:
    box: 'windows-server-2012r2'
    communicator: winrm
    ssh_disabled: true
    guest_type: windows
    roles:
      - windows_agent_standalone
```

Per platform you can override any of the options declared in [Defaults](#defaults-configuration).

### Instance Configuration

An instance within vagrant_dev represents an actual vm that you will be managing via the vagrant toolset.

Example instance configuration:

```yaml
instances:
  centos7a.example.demo:
    platform: centos7

  server2012r2a.example.demo:
    platform: server2012
```

Just like platforms, you can override any of the options declared in [Defaults](#defaults-configuration); however your instance level overides take precedence over platform level overrides.

### Role configuration
Roles represent inline_shell commands you'd like vagrant to run during provisioning steps.  The are configured in config/roles.yml and can be specified on an instance or platforms by passing in an array within yaml.

Example instance with roles configured:

```yaml
instances:
  centos7a.example.demo:
    platform: centos7
    roles:
      - posix_agent_standalone
```

Example snippet of a role:
```yaml
roles:
  posix_agent_standalone: |
    rpm -Uhv http://pm.puppetlabs.com/puppet-agent/2016.1.2/1.4.2/repos/el/7/PC1/x86_64/puppet-agent-1.4.2-1.el7.x86_64.rpm
```

## Puppet
> *Note:* You need to either install puppet using a role -OR- being using a box that has puppet pre-installed

Using the Puppet provisioner with this repo is simple, for Puppet < version 4 you can set at a minimum `puppet_enable: true`.  An example instance:

```yaml
instances:
  centos.example.lan:
    platform: centos7
    box: 'puppetlabs/centos-7.2-64-nocm'
    shared_folders:
      '/tmp': '/tmp/downloads'
    tcp_ports:
      '80': '10000'
    puppet_enable: 'true'
``` 
In this example puppet will look for a manifest called `default.pp` within a manifests directory at the base of your vagrant enviornment.

If using Puppet > 4, you'll need to also include a directory based setup for vagrant to use.  Example:

```yaml
instances:
  centos.example.lan:
    platform: centos7
    box: 'puppetlabs/centos-7.2-64-nocm'
    shared_folders:
      '/tmp': '/tmp/downloads'
    mem: 1024
    puppet_enable: 'true'
    puppet_environment: "production"
    puppet_environment_path: "."
```

In this example puppet will look for a manifest called `default.pp` within a manifests directory, that is within a `production` folder in your vagrant environment.

You can override any puppet settings found [here](https://www.vagrantup.com/docs/provisioning/puppet_apply.html), preface the setting name with `puppet_` at any level in the configuration(ie instance,platform,default).  Be sure to pay attention to the required data types when passing these in.
