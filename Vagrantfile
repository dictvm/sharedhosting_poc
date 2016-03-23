# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  config.vm.box = "uberspace/centos7"

  config.vm.define :selinux do |d|
    d.vm.hostname = 'selinux'
    d.vm.provision :ansible do |ansible|
      ansible.playbook = 'site.yml'
    end
  end

end
