# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

ip = '2.2.2.2'
cpus = 1
memory = 512 # in MB

ANSIBLE_PATH = __dir__ # absolute path to Ansible directory

# Set Ansible paths relative to Ansible directory
ENV['ANSIBLE_CONFIG'] = ANSIBLE_PATH
ENV['ANSIBLE_ROLES_PATH'] = File.join(ANSIBLE_PATH, 'vendor', 'roles')

site_file = File.join(ANSIBLE_PATH, 'group_vars', 'development', 'site.yml')

def fail_with_message(msg)
  fail Vagrant::Errors::VagrantError.new, msg
end

if File.exists?(site_file)
  site = YAML.load_file(site_file)['site']
  fail_with_message "No sites found in #{site_file}." if site.to_h.empty?
else
  fail_with_message "#{site_file} was not found. Please set `ANSIBLE_PATH` in your Vagrantfile."
end

if !Dir.exists?(ENV['ANSIBLE_ROLES_PATH']) && !Vagrant::Util::Platform.windows?
  fail_with_message "You are missing the required Ansible Galaxy roles, please install them with this command:\nansible-galaxy install -r requirements.yml"
end

Vagrant.configure("2") do |config|
  config.vm.box = 'ubuntu/trusty64'
  config.ssh.forward_agent = true

  # Fix for: "stdin: is not a tty"
  # https://github.com/mitchellh/vagrant/issues/1673#issuecomment-28288042
  config.ssh.shell = %{bash -c 'BASH_ENV=/etc/profile exec bash'}

  # Required for NFS to work
  config.vm.network :private_network, ip: ip, hostsupdater: 'skip'

  if Vagrant.has_plugin? 'vagrant-hostmanager'
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.aliases = site['host']
  else
    fail_with_message "vagrant-hostmanager missing, please install the plugin with this command:\nvagrant plugin install vagrant-hostmanager"
  end

  if !Vagrant.has_plugin? 'vagrant-bindfs'
    fail_with_message "vagrant-bindfs missing, please install the plugin with this command:\nvagrant plugin install vagrant-bindfs"
  else
    config.vm.synced_folder relative_path(site['local_path']), nfs_path(site['name']), type: 'nfs'
    config.bindfs.bind_folder nfs_path(site['name']), remote_site_path(site['name']), u: 'vagrant', g: 'www-data', o: 'nonempty'
  end

  config.vm.provision :ansible do |ansible|
    ansible.playbook = File.join(ANSIBLE_PATH, 'dev.yml')
    ansible.groups = {
      'web' => ['default'],
      'development' => ['default']
    }

    ansible.extra_vars = {'vagrant_version' => Vagrant::VERSION}
    if vars = ENV['ANSIBLE_VARS']
      extra_vars = Hash[vars.split(',').map { |pair| pair.split('=') }]
      ansible.extra_vars.merge(extra_vars)
    end
  end

  # Virtualbox settings
  config.vm.provider 'virtualbox' do |vb|
    vb.name = config.vm.hostname
    vb.customize ['modifyvm', :id, '--cpus', cpus]
    vb.customize ['modifyvm', :id, '--memory', memory]

    # Fix for slow external network connections
    vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
    vb.customize ['modifyvm', :id, '--natdnsproxy1', 'on']
  end

  # VMware Workstation/Fusion settings
  ['vmware_fusion', 'vmware_workstation'].each do |provider|
    config.vm.provider provider do |vmw, override|
      override.vm.box = 'puppetlabs/ubuntu-14.04-64-nocm'
      vmw.name = config.vm.hostname
      vmw.vmx['numvcpus'] = cpus
      vmw.vmx['memsize'] = memory
    end
  end

  # Parallels settings
  config.vm.provider 'parallels' do |prl, override|
    override.vm.box = 'parallels/ubuntu-14.04'
    prl.name = config.vm.hostname
    prl.cpus = cpus
    prl.memory = memory
  end

end

def relative_path(path)
  File.expand_path(path, ANSIBLE_PATH)
end

def nfs_path(site_name)
  "/vagrant-nfs-#{site_name}"
end

def remote_site_path(site_name)
  "/srv/www/#{site_name}/current"
end
