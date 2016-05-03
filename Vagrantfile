# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

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
    :box => 'fedora/23-cloud-base',
    :memory => 2048,
    :cpus => 2,
    :networks => [
      {
        :netid => :public_network,
        :auto_config => false,
        :dev => "virbr0",
        :type => "bridge",
      },
    ],
    :disks => [
      {
        :size => 2, #gigabytes
        :parts => [
          {
            :fs => "xfs",
            :mount => "/data",
            :name => "data",
            :size => "100%",
          },
        ],
      },
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
  begin
    custom_settings = YAML::load_file f
    if custom_settings == false then raise end
  rescue
    retry
  end
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
# Derive virtual disk device names and partition numbers
#

driveletters = ('b'..'z').to_a

vms_common[:disks].each_with_index do |disk,disk_num|
  disk[:num] = disk_num
  disk[:dev_names] = {
    :libvirt => "vd#{driveletters[disk[:num]]}",
  }
  disk[:parts].each_with_index do |part,part_num|
    part[:num] = part_num + 1
  end
end

#==============================================================================
#
# Define required software for groups
#

group_defs = {
  :ctdb_servers => {
    :install_pkgs => " ctdb",
    :services => [ "ctdb" ],
  },
  :samba_servers => {
    :install_pkgs => " samba samba-winbind samba-winbind-clients",
    :services => [],
  },
  :gluster_servers => {
    :install_pkgs => " glusterfs-server glusterfs-client",
    :services => [ "glusterd" ],
  },
 :clients => {
    :install_pkgs => " cifs-utils glusterfs-fuse",
 },
}
if gluster[:setup_gluster]
  group_defs[:samba_servers][:install_pkgs] << " samba-vfs-glusterfs"
end
if not ctdb[:setup_ctdb]
  group_defs[:samba_servers][:services].push "winbind"
  group_defs[:samba_servers][:services].push "smb"
  group_defs[:samba_servers][:services].push "nmb"
end

#==============================================================================
#
# active_vms - Keep track of currently running VMs, since vagrant won't tell
#              us directly.
#

active_vms = []

f = File.join(projectdir, 'active_vms.yaml')

if File.exists?(f)
  begin
    active_vms = YAML::load_file f
    if active_vms == false then raise end
  rescue
    retry
  end
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
      else
        groups[name][i] = node
      end
    end
  end
end
if ad[:setup_ad] and not groups.keys.include? "ad_server"
  groups[:ad_server] = group[:samba_servers][0]
end

if ENV['VAGRANT_LOG'] == 'debug'
  p "groups: #{groups}"
end

#==============================================================================
#
# Collect packages to install and services to run
#

install_pkgs = {}
services = {}
if active_vms.length > 0
  active_vms.each do |name|
    install_pkgs[name] = "yum python python-dnf python-simplejson yum libselinux-python xfsprogs gnupg "
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
      if group_defs and group_defs[name]
        install_pkgs[node] << group_defs[name][:install_pkgs] if group_defs[name][:install_pkgs]
        services[node].push group_defs[name][:services] if group_defs[name][:services]
      end
      if group_vars and group_vars[name]
        install_pkgs[node] << " " + group_vars[name][:install_pkgs] if group_vars[name][:install_pkgs]
        services[node].push group_vars[name][:services] if group_vars[name][:services]
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
      node.vm.provider :libvirt do |domain|
        domain.memory = vms_common[:memory]
        domain.cpus = vms_common[:cpus]
      end

      if vms_common[:disks]
        vms_common[:disks].each do |disk|
          node.vm.provider :libvirt do |lv|
            lv.storage :file, :size => "#{disk[:size]}G", :device => "#{disk[:dev_names][:libvirt]}"
            disk[:dev] = disk[:dev_names][:libvirt]
          end
        end
      end

      if vms_common[:networks]
        vms_common[:networks].each do |net|
          netid = net[:netid]
          netopts = net.except(:netid)
          i = vms_common[:networks].index(net)
          if machine[:networks] and i < machine[:networks].length
            netopts.merge!(machine[:networks][i])
          end
          node.vm.network netid, netopts
        end
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
      if ENV['RUN']
        playbooks.push(ENV['RUN'])
      else
        playbooks.push("playbooks/raw-f23.yml")
        custom_pre_provision = ENV['CUSTOM_PRE'] ? ENV['CUSTOM_PRE'] : "playbooks/custom_pre.yml"
        if File.exists?(custom_pre_provision)
          playbooks.push(custom_pre_provision)
        end
        playbooks.push("playbooks/samba-cluster.yml")
        custom_post_provision = ENV['CUSTOM_POST'] ? ENV['CUSTOM_POST'] : "playbooks/custom_post.yml"
        if File.exists?(custom_post_provision)
          playbooks.push(custom_post_provision)
        end
      end
      playbooks.each do |playbook|
        node.vm.provision "ansible" do |ansible|
          if ENV['ANSIBLE_DEBUG']
            ansible.verbose = ENV['ANSIBLE_DEBUG']
          end
          ansible.playbook = playbook
          ansible.groups = {}
          groups.each do |name,group|
            ansible.groups[name.to_s] = group
          end
          ansible.extra_vars = {
            "extra_disks" => vms_common[:disks],
            "vips"        => vms_common[:virtual_ips],
            "samba"       => samba,
            "ctdb"        => ctdb,
            "ad"          => ad,
            "gluster"     => gluster,
          }
          if ad[:setup_ad]
            ansible.extra_vars['ad_passwd'] = ad_passwd
          end
          if vms_common[:extra_vars]
            ansible.extra_vars.merge! vms_common[:extra_vars]
          end
          if ENV['EXTRA_VARS']
            ansible.extra_vars.merge! eval ENV['EXTRA_VARS']
          end
          ansible.limit = "all"
        end
      end
    end
  end
end
