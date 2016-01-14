# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'lxc'

VAGRANTFILE_API_VERSION = "2"

require 'yaml'
require 'io/console'

projectdir = File.expand_path File.dirname(__FILE__)

#==============================================================================
#
# Default VM settings
#

settings = {
  :vms => [
    { :name => 'node1', },
    { :name => 'node2', },
    { :name => 'node3', },
    { :name => 'node4', },
  ],
  :vms_common => {
    :box => 'obnox/fedora23-64-lxc',
    :memory => 2048,
    :cpus => 2,
    :networks => [
      {
        :type => 'veth',
        :link => 'virbr1',
        :name => 'eth0',
        :flags => 'up',
      },
    ],
    :synced_folders => [
      { :src => "lock/", :dest => "/shared/lock" },
    ],
    :virtual_ips => [
      { :ip => "192.168.121.111" },
      { :ip => "192.168.121.112" },
    ],
  },
  :groups => {
    :samba_servers => [ "all" ],
    :gluster_servers => [ "all" ],
  },
  :samba => {
    :setup_samba => true,
    :config => nil,
  },
  :ctdb => {
    :setup_ctdb => true,
    :config => nil,
  },
  :ad => {
    :setup_ad => false,
    :domain => "domain.com",
    :dns => "0.0.0.0",
  },
  :gluster => {
    :setup_gluster => true,
    :bricks_dir => "/data/bricks",
    :volumes => [
        { name: 'share' },
        { name: 'ctdb', replica: "n", mount: "/shared/lock" },
      ],
  },
}

#==============================================================================
#
# Load (if present) and write out custom settings
#

custom_settings = {}

f = File.join(projectdir, 'vagrant.yaml')

if File.exists?(f)
  custom_settings = YAML::load_file f
else
  File.open(f, 'w') do |file|
    file.write settings.to_yaml
  end
  puts "Wrote initial config: [ #{f} ]"
  puts "Please verify settings and run your command again."
  exit
end

settings.merge!(custom_settings)

File.open(f, 'w') do |file|
  file.write settings.to_yaml
end

vms        = settings[:vms]
vms_common = settings[:vms_common]
groups     = settings[:groups]
group_vars = settings[:group_vars]
samba      = settings[:samba]
ctdb       = settings[:ctdb]
ad         = settings[:ad]
gluster    = settings[:gluster]

#==============================================================================
#
# Define required software for groups
#

group_defs = {
  :samba_servers => {
    :pkgs => " samba samba-winbind",
    :services => [],
  },
}
if ctdb[:setup_ctdb]
  group_defs[:samba_servers][:pkgs] << "ctdb"
  group_defs[:samba_servers][:services].push "ctdb"
else
  group_defs[:samba_servers][:services].push "smb"
  group_defs[:samba_servers][:services].push "nmb"
  group_defs[:samba_servers][:services].push "winbind"
end

#==============================================================================
#
# active_vms - Keep track of currently running VMs, since vagrant won't tell
#              us directly.
#

active_vms = []

f = File.join(projectdir, 'active_vms.yaml')

if File.exists?(f)
  active_vms = YAML::load_file f
end

if ARGV[0] == "up"
  cmd_names = ARGV.drop(1).delete_if { |x| x.start_with?("-") or active_vms.include?(x) }
  if cmd_names.length > 0 then
    active_vms.push(*cmd_names)
  else
    vms.each do |x|
      if not active_vms.include?(x[:name])
        active_vms.push x[:name]
      end
    end
  end
elsif ARGV[0] == "destroy" or ARGV[0] == "halt"
  cmd_names = ARGV.drop(1).delete_if { |x| x.start_with?("-") or not active_vms.include?(x) }
  if cmd_names.length > 0 then
    active_vms.delete_if { |x| cmd_names.include?(x) }
  else
    active_vms = []
  end
end

File.open(f, 'w+') do |file|
  file.write active_vms.to_yaml
end

if ENV['VAGRANT_LOG'] == 'debug'
  p "active_vms: #{active_vms}"
end

#==============================================================================
#
# Build group listings
#

groups.each do |name,group|
  if group.include? "all"
    groups[name] = active_vms
  else
    group.each_with_index do |node,i|
      case node
      when "first"
        groups[name][i] = active_vms[0]
      when "last"
        groups[name][i] = active_vms[-1]
      when "not first"
        groups[name] = active_vms.count > 1 ? active_vms[1..-1] : [ active_vms[0] ]
      when "not last"
        groups[name] = active_vms.count > 1 ? active_vms[0..-2] : [ active_vms[0] ]
      when node.is_a?(Integer)
        groups[name][i] = active_vms[node]
      end
    end
  end
end
if ad[:setup_ad] and not groups.keys.include? "ad_server"
  groups[:ad_server] = group[:samba_servers][0]
end

#==============================================================================
#
# Collect packages to install and services to run
#

install_pkgs = {}
services = {}
if active_vms.length > 0
  active_vms.each do |name|
    install_pkgs[name] = "yum python python-simplejson yum libselinux-python xfsprogs gnupg "
    if vms_common[:install_pkgs]
      install_pkgs[name] << " " + vms_common[:install_pkgs]
    end

    services[name] = []
    if vms_common[:services]
      services[name].push vms_common[:services]
    end
  end
  groups.each do |name,group|
    group.each do |node|
      install_pkgs[node] << group_defs[name][:pkgs]
      if group_vars and group_vars[name] and group_vars[name][:install_pkgs]
        install_pkgs[node] << " " + group_vars[name][:install_pkgs]
      end

      services[node].push group_defs[name][:services]
      if group_vars and group_vars[name] and group_vars[name][:services]
        services[node].push group_vars[name][:services]
      end
    end
  end
  vms.each do |vm|
    if vm['install_pkgs']
      install_pkgs[name] << " " + vm['install_pkgs']
    end
    if vm['services']
      services[name].push vm[:services]
    end
  end
end

#==============================================================================
#
# Vagrant config
#

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
#  if Vagrant.has_plugin?("vagrant-cachier")
#    config.cache.scope = :box
#    config.cache.auto_detect = false
#    config.cache.enable :yum
#  end

  config.ssh.insert_key = false

  vms.each do |machine|
    config.vm.define machine[:name] do |node|
      node.vm.box = vms_common[:box]
      node.vm.provider :lxc do |lxc|
        lxc.container_name = machine[:container_name]

        vms_common[:networks].each_with_index do |net,i|
          netopts = net
          if machine[:networks] and i < machine[:networks].length
            netopts.merge!(machine[:networks][i])
          end
          netopts.each_pair do |key,val|
            if not net[key] == ''
             lxc.customize "network.#{key}", val
            end
          end
        end
      end

      if vms_common[:networks]
      end

      if vms_common[:sync_folders]
        vms_common[:sync_folders].each do |sync|
          src = sync[:src]
          dest = sync[:dest]
          syncopts = sync.except(:src, :dest)
          node.vm.synced_folder src, dest, syncopts
        end
      end
      if machine[:sync_folders]
        machine[:sync_folders].each do |sync|
          src = sync[:src]
          dest = sync[:dest]
          syncopts = sync.except(:src, :dest)
          node.vm.synced_folder src, dest, syncopts
        end
      end

    end
  end

  if active_vms.length > 0 then
    config.vm.define active_vms[0], primary: true do |node|
      if ad[:setup_ad]
        print "AD Administrator password: "
        ad_passwd = STDIN.noecho(&:gets)
      end

      active_vms.each do |node|
        host_vars = {}
        host_vars['install_pkgs'] = install_pkgs[node]
        host_vars['services'] = services[node]
        File.open('playbooks/host_vars/' + node.to_s, 'w+') do |file|
          file.write host_vars.to_yaml
        end
      end

      playbooks = []
      playbooks.push("playbooks/raw-f23.yml")
      custom_provision = "playbooks/custom.yml"
      if File.exists?(custom_provision)
        playbooks.push(custom_provision)
      end
      playbooks.push("playbooks/samba-cluster.yml")
      playbooks.each do |playbook|
        node.vm.provision "ansible" do |ansible|
#          ansible.verbose = "vvv"
          ansible.playbook = playbook
          ansible.groups = {}
          groups.each do |name,group|
            ansible.groups[name.to_s] = group
          end
          ansible.extra_vars = {
            "vips"        => vms_common[:virtual_ips],
            "samba"       => samba,
            "ctdb"        => ctdb,
            "ad"          => ad,
            "gluster"     => gluster,
          }
          if ad[:setup_ad]
            ansible.extra_vars['ad_passwd'] = ad_passwd
          end
          ansible.limit = "all"
        end
      end
    end
  end
end
