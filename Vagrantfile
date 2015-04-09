# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'ipaddr'

unless Vagrant.has_plugin?("vagrant-hostmanager")
  raise 'vagrant-hostmanager is not installed! run "vagrant plugin install vagrant-hostmanager" to fix'
end

VAGRANTFILE_API_VERSION = "2"

class IpAssigner

  def self.next_ip(current_ip)
    IPAddr.new(current_ip).succ.to_s
  end

  def self.generate(start_ip, number_of_addresses)
    number_of_addresses.times.inject([start_ip]) { |acc, i|
      nekst = next_ip(acc.last)
      acc + [nekst]
    }
  end

end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = 'hfm4/centos7'
  config.ssh.insert_key = false
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  accept_oracle_licence = true # set to false if you don't agree (will not install Java8 for you)

  private_network_begin =  "192.168.5.100" # private ip will start incrementing from this

  # ZK config
  zk_vm_memory_mb = 256
  zk_port = 2181

  # Spark master config
  spark_master_memory_mb = 1024
  spark_master_work_port = 7077
  spark_master_ui_port = 8080

  # Spark worker configs
  spark_worker_memory_mb = 1024
  spark_worker_work_port = 7178
  spark_worker_ui_port = 8181

  # < ------- These need to be set in group vars if using Ansible w/o Vagrant -------

  # The follwing variables will need to be passed manually if you want to use the Ansible
  # playbooks w/o Vagrant. They are set in this Vagrantfile in this manner because it allows us to easily
  # increase or decrease the cluster sizes.

  # Note that zk_id must be unique for each host in the cluster. It should ideally not change
  # throughout the lifetime of the Zookeeper installation on a given machine.
  zk_cluster_info = {
    'zk-node-1' => { :zk_id => 1 },
    'zk-node-2' => { :zk_id => 2 },
    'zk-node-3' => { :zk_id => 3 }
  }

  spark_cluster_masters = 2

  spark_cluster_workers = 2

  ## ------- These need to be set in group vars if using Ansible w/o Vagrant ------- >

  # Helper to make new ips
  zk_ips = IpAssigner.generate(
    private_network_begin,
    zk_cluster_info.size)

  spark_master_ips = IpAssigner.generate(
    IpAssigner.next_ip(zk_ips.last || private_network_begin),
    spark_cluster_masters)

  spark_worker_ips = IpAssigner.generate(
    IpAssigner.next_ip(spark_master_ips.last || private_network_begin),
    spark_cluster_workers)

  zk_cluster = Hash[zk_cluster_info.map.with_index { |(k, v), idx|
    [k, v.merge(
      :ip => zk_ips[idx],
      :memory => zk_vm_memory_mb,
      :forwarded_ports => {
        zk_port => zk_port + idx # guest port => host port
      })]
  }]

  spark_master_cluster = Hash[spark_cluster_masters.times.map { |i|
    [
      "spark-master-node-#{i + 1}",
      {
        :ip => spark_master_ips[i],
        :memory => spark_master_memory_mb,
        :forwarded_ports => {
          spark_master_work_port => spark_master_work_port + i,
          spark_master_ui_port => spark_master_ui_port + i
        }
      }
    ]
  }]

  spark_worker_cluster = Hash[spark_cluster_workers.times.map { |i|
    [
      "spark-worker-node-#{i + 1}",
      {
        :ip => spark_worker_ips[i],
        :memory => spark_worker_memory_mb,
        :forwarded_ports => {
          spark_worker_work_port => spark_worker_work_port + i,
          spark_worker_ui_port => spark_worker_ui_port + i
        }
      }
    ]
  }]

  total_cluster = zk_cluster.merge(spark_master_cluster).merge(spark_worker_cluster)

  total_cluster.each_with_index do |(short_name, info), idx|

    config.vm.define short_name do |host|
      info[:forwarded_ports].each { |(client_port, host_port)|
        host.vm.network :forwarded_port, guest: client_port, host: host_port
      }
      host.vm.network :private_network, ip: info[:ip]
      host.vm.hostname = short_name
      host.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", info[:memory]]
      end

      # This allows us to provision everything in one go, in parallel.
       if idx == (total_cluster.size - 1)
         host.vm.provision :ansible do |ansible|
           ansible.playbook = "site.yml"
           ansible.groups = {
             "zk" => zk_cluster.keys,
             "spark-masters" => spark_master_cluster.keys,
             "spark-workers" => spark_worker_cluster.keys,
             "spark:children" => [ "spark-masters", "spark-workers" ]
           }
           ansible.verbose = 'v'
           ansible.sudo = true
           ansible.limit = 'all' # otherwise, Ansible only runs on the current host...
           ansible.extra_vars = {
             accept_oracle_licence: accept_oracle_licence,
             vagrant_zk_client_port: zk_port,
             vagrant_zk_cluster_info: zk_cluster_info,
             vagrant_zk_cluster_hostnames: zk_cluster.keys,
             vagrant_spark_master_hostnames: spark_master_cluster.keys,
             vagrant_spark_master_work_port: spark_master_work_port,
             vagrant_spark_master_ui_port: spark_master_ui_port,
             vagrant_spark_worker_work_port: spark_worker_work_port,
             vagrant_spark_worker_ui_port: spark_worker_ui_port
           }
         end
       end

    end

  end

end
