require 'fileutils'
Vagrant.require_version '>= 2.2.2'

# file operations needs to be relative to this file
VAGRANT_ROOT = File.dirname(File.expand_path(__FILE__))

# directory that will contain VDI files
VAGRANT_DISKS_DIRECTORY = 'disks'
# config.vm.synced_folder '/host/path', '/guest/path', SharedFoldersEnableSymlinksCreate: false

# controller definition
VAGRANT_CONTROLLER_NAME = 'Virtual I/O Device SCSI controller'
VAGRANT_CONTROLLER_TYPE = 'virtio-scsi'

# define disks
# The format is filename, size (GB), port (see controller docs)
local_disks = [
  { filename: 'disk1', size: 100, port: 5 },
  { filename: 'disk2', size: 100, port: 6 },
  { filename: 'disk3', size: 50, port: 25 }
]

Vagrant.configure(2) do |config|
  config.vm.define 'ananas-test' do
    config.vm.box = 'ubuntu/focal64'
    config.vm.network 'public_network', ip: '192.168.1.40'
    config.ssh.insert_key = false

    disks_directory = File.join(VAGRANT_ROOT, VAGRANT_DISKS_DIRECTORY)

    # create disks before "up" action
    config.trigger.before :up do |trigger|
      trigger.name = 'Create disks'
      trigger.ruby do
        FileUtils.mkdir_p(disks_directory) unless File.directory?(disks_directory)
        local_disks.each do |local_disk|
          local_disk_filename = File.join(disks_directory, "#{local_disk[:filename]}.vdi")
          unless File.exist?(local_disk_filename)
            puts "Creating \"#{local_disk[:filename]}\" disk"
            system("vboxmanage createmedium --filename #{local_disk_filename} --size #{local_disk[:size] * 1024} --format VDI")
          end
        end
      end
    end

    # create storage controller on first run
    unless File.directory?(disks_directory)
      config.vm.provider 'virtualbox' do |storage_provider|
        storage_provider.customize ['storagectl', :id, '--name', VAGRANT_CONTROLLER_NAME, '--add',
                                    VAGRANT_CONTROLLER_TYPE, '--hostiocache', 'off']
      end
    end

    # attach storage devices
    config.vm.provider 'virtualbox' do |storage_provider|
      local_disks.each do |local_disk|
        local_disk_filename = File.join(disks_directory, "#{local_disk[:filename]}.vdi")
        unless File.exist?(local_disk_filename)
          storage_provider.customize ['storageattach', :id, '--storagectl', VAGRANT_CONTROLLER_NAME, '--port',
                                      local_disk[:port], '--device', 0, '--type', 'hdd', '--medium', local_disk_filename]
        end
      end
      storage_provider.memory = 12_288
      storage_provider.cpus = 2
    end

    # cleanup after "destroy" action
    config.trigger.after :destroy do |trigger|
      trigger.name = 'Cleanup operation'
      trigger.ruby do
        # the following loop is now obsolete as these files will be removed automatically as machine dependency
        local_disks.each do |local_disk|
          local_disk_filename = File.join(disks_directory, "#{local_disk[:filename]}.vdi")
          if File.exist?(local_disk_filename)
            puts "Deleting \"#{local_disk[:filename]}\" disk"
            system("vboxmanage closemedium disk #{local_disk_filename} --delete")
          end
        end
        FileUtils.rmdir(disks_directory) if File.exist?(disks_directory)
      end
    end

    config.vm.provision 'ansible_local' do |ansible|
      ansible.compatibility_mode = '2.0'
      ansible.galaxy_role_file = 'requirements.yml'
      ansible.inventory_path = 'tests/inventories/integration_testing/inventory'
      ansible.playbook = 'playbook.yml'
      ansible.become = true
    end
  end
end
