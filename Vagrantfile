# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'
cwd =  File.dirname(__FILE__)

def missing_setting(setting,machine)
  abort("ERROR: Missing setting [#{setting}] for machine [#{machine}]")
end

def get_settings(platforms,settings,instance,setting)
  # Instance Override
  if settings['instances'][instance][setting] != nil
    return settings['instances'][instance][setting]

  # Platform Setting
  elsif platforms[ settings['instances'][instance]['platform'] ][setting] != nil
    return platforms[ settings['instances'][instance]['platform'] ][setting]

  # Global Instance Default
  elsif settings['instance_defaults'][setting] != nil
    return settings['instance_defaults'][setting]

  # Global Default
  elsif settings['defaults'][setting] != nil
    return settings['defaults'][setting]

  # Fail out, required setting is missing
  else
    # If this is a required setting, fail.
    if ! ['roles','guest_type','shared_folders','tcp_ports','udp_ports'].include?(setting) && setting !~ /^puppet.+/
      missing_setting(setting,instance)
    end
  end
end

# Load Config
settings              = YAML.load_file cwd + '/config/default.yml'
instance_raw          = YAML.load_file cwd + '/config/instances.yml'
role_raw              = YAML.load_file cwd + '/config/roles.yml'
platform_raw          = YAML.load_file cwd + '/config/platforms.yml'

instances             = instance_raw['instances']
settings['instances'] = instances
roles                 = role_raw['roles']
platforms             = platform_raw['platforms']


Vagrant.configure(2) do |config|
    settings['instances'].keys.each {|i|
      config.vm.define i, primary: get_settings(platforms,settings,i,'primary'), autostart: get_settings(platforms,settings,i,'autoup') do |vmconfig|
        vmconfig.vm.hostname              = settings['instances'][i]['name']
        vmconfig.vm.box                   = get_settings(platforms,settings,i,'box')
        vmconfig.vm.communicator          = get_settings(platforms,settings,i,'communicator')
        vmconfig.winrm.username           = get_settings(platforms,settings,i,'winrm_username')
        vmconfig.winrm.password           = get_settings(platforms,settings,i,'winrm_password')
        vmconfig.ssh.username             = get_settings(platforms,settings,i,'ssh_username')
        vmconfig.ssh.pty                  = true

        if get_settings(platforms,settings,i,'guest_type') != nil
          vmconfig.vm.guest = get_settings(platforms,settings,i,'guest_type')
          if get_settings(platforms,settings,i,'guest_type') == 'windows'
            vmconfig.vm.network :forwarded_port, host: 33389, guest: 3389, id: 'rdp', auto_correct: true
          end
        end

        vmconfig.vm.provider "virtualbox" do |v|
          v.memory = get_settings(platforms,settings,i,'mem')
          v.cpus   = get_settings(platforms,settings,i,'cpus')
          v.customize ["modifyvm", :id, "--ioapic", "on"]
        end

        vmconfig.vm.provision :hosts do |provisioner|
          provisioner.sync_hosts = true
          provisioner.autoconfigure = true
          provisioner.exports = {
            'global' => [
              ['@facter_ipaddress', ['@vagrant_hostnames']],
            ],
          }
          provisioner.imports = ['global']
        end

        vmroles = get_settings(platforms,settings,i,'roles')
        if vmroles
          vmroles.each {|r|
            if roles.key?(r)
              vmconfig.vm.provision "shell",
                inline: roles[r]
            else
              abort("Error: VM #{i} configured role #{r} is not valid! Existing...")
            end
          }
        end

        # Setup forwarded TCP ports
        get_settings(platforms,settings,i,'tcp_ports').each {|vport,hport|
          vmconfig.vm.network "forwarded_port", guest: vport, host: hport, protocol: 'tcp'
        } if get_settings(platforms,settings,i,'tcp_ports')

        # Setup forwarded UDP ports
        get_settings(platforms,settings,i,'udp_ports').each {|vport,hport|
          vmconfig.vm.network "forwarded_port", guest: vport, host: hport, protocol: 'udp'
        } if get_settings(platforms,settings,i,'udp_ports')

        # Setup sync'd folders
        get_settings(platforms,settings,i,'shared_folders').each {|host_folder,guest_folder|
          vmconfig.vm.synced_folder host_folder, guest_folder
        } if get_settings(platforms,settings,i,'shared_folders')

        # Setup Puppet Provisioner
        if get_settings(platforms,settings,i,'puppet_enable') == 'true'
          vmconfig.vm.provision "puppet" do |puppet|
            puppet.facter             = get_settings(platforms,settings,i,'puppet_facter')             if get_settings(platforms,settings,i,'puppet_facter')
            puppet.hiera_config_path  = get_settings(platforms,settings,i,'puppet_hiera_config_path')  if get_settings(platforms,settings,i,'puppet_hiera_config_path')
            puppet.manifest_file      = get_settings(platforms,settings,i,'puppet_manifest_file')      if get_settings(platforms,settings,i,'puppet_manifest_file')
            puppet.manifests_path     = get_settings(platforms,settings,i,'puppet_manifests_path')     if get_settings(platforms,settings,i,'puppet_manifests_path')
            puppet.module_path        = get_settings(platforms,settings,i,'puppet_module_path')        if get_settings(platforms,settings,i,'puppet_module_path')
            puppet.environment        = get_settings(platforms,settings,i,'puppet_environment')        if get_settings(platforms,settings,i,'puppet_environment')
            puppet.environment_path   = get_settings(platforms,settings,i,'puppet_environment_path')   if get_settings(platforms,settings,i,'puppet_environment_path')
            puppet.options            = get_settings(platforms,settings,i,'puppet_options')            if get_settings(platforms,settings,i,'puppet_options')
            puppet.synced_folder_type = get_settings(platforms,settings,i,'puppet_synced_folder_type') if get_settings(platforms,settings,i,'puppet_synced_folder_type')
            puppet.temp_dir           = get_settings(platforms,settings,i,'puppet_temp_dir')           if get_settings(platforms,settings,i,'puppet_temp_dir')
            puppet.working_directory  = get_settings(platforms,settings,i,'puppet_working_directory')  if get_settings(platforms,settings,i,'puppet_working_directory')
          end
        end
      end
    }
end
