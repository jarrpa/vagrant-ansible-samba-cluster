# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

VAGRANTFILE_API_VERSION = "2"

require 'yaml'

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

vms = settings[:vms]
vms_common = settings[:vms_common]
samba = settings[:samba]
ctdb = settings[:ctdb]
ad = settings[:ad]

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
    vms.each { |x| active_vms << x[:name] }
  end
elsif ARGV[0] == "destroy" or ARGV[0] == "halt"
  cmd_names = ARGV.drop(1).delete_if { |x| x.start_with?("-") or not active_vms.include?(x) }
  if cmd_names.length > 0 then
    active_vms.delete_if { |x| cmd_names.include?(x) }
  else
    active_vms = []
  end
end

File.open(f, 'w') do |file|
  file.write active_vms.to_yaml
end

if ENV['VAGRANT_LOG'] == 'debug'
  p "active_vms: #{active_vms}"
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

    end
  end

  if active_vms.length > 0 then
    config.vm.define active_vms[0], primary: true do |node|
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
          ansible.groups = {
            "storage_servers" => active_vms,
          }
          ansible.extra_vars = {
            "extra_disks" => vms_common[:disks],
            "vips"        => vms_common[:virtual_ips],
            "samba"       => samba,
            "ctdb"        => ctdb,
            "ad"          => ad,
          }
          ansible.limit = "all"
        end
      end
    end
  end
end
