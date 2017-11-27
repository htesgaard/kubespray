This page contains my onw notes on using kubespray on Vagrant with GlusterFS

# Preqs

Netaddr
```
$ pip install netaddr
$ which netaddr
  /Users/helge/.pyenv/shims/netaddr
```

# Usage

Checkout this project and run `vagrant up`

# Provision glusterfs from commandline
```

/Users/helge/.pyenv/versions/2.7.14/envs/tools2/bin/python2.7 /Users/helge/.pyenv/versions/tools2/bin/ansible-playbook --connection=ssh --timeout=30 --limit=all --inventory-file=./.vagrant/provisioners/ansible/inventory --become --forks=3 --flush-cache -vvvv ./contrib/network-storage/glusterfs/glusterfs.yml
```

# Custom provisioning from the commandline
Regular provision from commandline equal to `vagrant provision`
```
/Users/helge/.pyenv/versions/2.7.14/envs/tools2/bin/python2.7 /Users/helge/.pyenv/versions/tools2/bin/ansible-playbook --connection=ssh --timeout=30 --limit=all --inventory-file=./.vagrant/provisioners/ansible/inventory --become --forks=3 --flush-cache -vvvv cluster.yml
```


# My changes

## GlusterFS 
To get some persistence for GlusterFS the Vagrant file has been tweaked a bit.

```diff
diff --git a/Vagrantfile b/Vagrantfile
index 40109f9b..de940479 100644
--- a/Vagrantfile
+++ b/Vagrantfile
@@ -106,10 +106,16 @@ Vagrant.configure("2") do |config|
         config.vm.synced_folder src, dst
       end
 
+      file_to_disk = "./.vagrant/disks/glusterfs_#{i}.vdi"
       config.vm.provider :virtualbox do |vb|
         vb.gui = $vm_gui
         vb.memory = $vm_memory
         vb.cpus = $vm_cpus
+        unless File.exist?(file_to_disk)
+            vb.customize ['createhd', '--filename', file_to_disk, '--size', 500 * 1024]
+        end
+        vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', file_to_disk]
+
       end
 
       ip = "#{$subnet}.#{i+100}"
@@ -118,7 +124,7 @@ Vagrant.configure("2") do |config|
         "bootstrap_os": SUPPORTED_OS[$os][:bootstrap_os],
         "local_release_dir" => $local_release_dir,
         "download_run_once": "False",
-        "kube_network_plugin": $network_plugin
+        "disk_volume_device_1": "/dev/sdb",
       }
 
       config.vm.network :private_network, ip: ip
@@ -143,14 +149,16 @@ Vagrant.configure("2") do |config|
           ansible.sudo = true
           ansible.limit = "all"
           ansible.host_key_checking = false
-          ansible.raw_arguments = ["--forks=#{$num_instances}", "--flush-cache"]
+          ansible.raw_arguments = ["--forks=#{$num_instances}", "--flush-cache", "-vvvv"]
           ansible.host_vars = host_vars
           #ansible.tags = ['download']
           ansible.groups = {
             "etcd" => ["#{$instance_name_prefix}-0[1:#{$etcd_instances}]"],
             "kube-master" => ["#{$instance_name_prefix}-0[1:#{$kube_master_instances}]"],
             "kube-node" => ["#{$instance_name_prefix}-0[1:#{$kube_node_instances}]"],
+            "gfs-cluster" => ["#{$instance_name_prefix}-0[1:#{$kube_node_instances}]"],
             "k8s-cluster:children" => ["kube-master", "kube-node"],
+            "network-storage:children:children" => ["gfs-cluster"],
           }
         end
       end

```

## DNS resolution

To proxy DNS resolution on the guests to the host resolver, this tweak has been done to the Vagrantfile
```diff
diff --git a/Vagrantfile b/Vagrantfile
index 2eefbfdb..c839d709 100644
--- a/Vagrantfile
+++ b/Vagrantfile
@@ -115,6 +115,12 @@ Vagrant.configure("2") do |config|
             vb.customize ['createhd', '--filename', file_to_disk, '--size', 500 * 1024]
         end
         vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', file_to_disk]
+
+        # Configure NAT Network to proxy DNS requests to the host DNS resolver
+        # --natdnshostresolver<1-N> on|off: This setting makes the NAT engine use the host's resolver mechanisms to handle DNS requests. Please see Section 9.11.5, “Enabling DNS proxy in NAT mode” for detailsx).
+        # https://www.virtualbox.org/manual/ch09.html#nat-adv-dns
+        # https://gist.github.com/jedi4ever/5657094
+        vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
       end
 
       ip = "#{$subnet}.#{i+100}"
```