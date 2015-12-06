Vagrant.configure("2") do |config|

  # Custom modern.ie packaged box
  config.vm.box = "win7-ie11-custom"

  # big timeout since windows boot is very slow
  config.vm.boot_timeout = 500

  # Port forward WinRM and RDP
  config.vm.network "forwarded_port", guest: 3389, host: 3389, id: "rdp", auto_correct: true
  config.vm.network :forwarded_port, guest: 5985, host: 5985, id: "winrm", auto_correct: true

  # winrm config, uses modern.ie default user/password. If other credentials are used must be changed here
  config.vm.communicator = "winrm"
  config.winrm.host = "localhost"
  config.winrm.username = "IEUser"
  config.winrm.password = "Passw0rd!"
  config.vm.guest = :windows

  # Shared folders
  config.vm.synced_folder "/home/marvin/Dropbox", "/Dropbox"
  config.vm.synced_folder "/home/marvin/crap", "/crap"

  config.vm.provider "virtualbox" do |vb|
    vb.gui = true
    vb.customize ["modifyvm", :id, "--memory", "1024"]
    vb.customize ["modifyvm", :id, "--vram", "128"]
    vb.customize ["modifyvm", :id,  "--cpus", "2"]
    vb.customize ["modifyvm", :id, "--ioapic", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 10000]
    vb.customize ['modifyvm', :id, '--clipboard', 'bidirectional']
    vb.customize ["modifyvm", :id, "--usb", "on"]
    vb.customize ["modifyvm", :id, "--usbehci", "on"]
  end
end
