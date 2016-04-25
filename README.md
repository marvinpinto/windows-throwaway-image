# Throwaway Windows Vagrant Images

I am primarily a Linux user but having access to a Windows machine every now
and then comes in very handy! One of the neat things that I have got very used
to in Linux is the ability to destroy Linux VMs and containers whenever I am
done with them.

Thanks to the lovely work from the folks at [modern.ie](http://modern.ie), we
can now bring up Windows virtual machines using Vagrant!


## How does this work?

The gist of it is that you need to create a customized (packaged) vagrant box
for yourself, based off a modern.ie image. The reason for this is because the
(Windows 7) modern.ie images are missing a bit of base functionality that make
is a bit harder for vagrant to interact with.

After you create your own customized image, bringing it up and destroying
instances of it becomes just another vagrant operation, yay!


## Pre-requisites

The host machine must have:

- [Vagrant](http://www.vagrantup.com/downloads.html)
- [VirtualBox](https://www.virtualbox.org/)


## Creating a customized image

#### Installation

1. Spin up a base Windows 7 vagrant image

  ```bash
  mkdir -p /tmp/win7-custom-image
  cat << EOF > /tmp/win7-custom-image/Vagrantfile
  Vagrant.configure("2") do |config|

    # Other box urls: https://www.bram.us/2014/09/24/modern-ie-vagrant-boxes
    config.vm.box = "modern.ie/win7-ie11"
    config.vm.box_url = 'http://aka.ms/vagrant-win7-ie11'

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
      vb.customize ["modifyvm", :id, "--usbxhci", "on"]
    end
  end
  EOF
  cat << EOF > /tmp/win7-custom-image/winrm.bat
  @echo off
  set WINRM_EXEC=call %SYSTEMROOT%\System32\winrm
  %WINRM_EXEC% quickconfig -q
  %WINRM_EXEC% set winrm/config/winrs @{MaxMemoryPerShellMB="300"}
  %WINRM_EXEC% set winrm/config @{MaxTimeoutms="1800000"}
  %WINRM_EXEC% set winrm/config/client/auth @{Basic="true"}
  %WINRM_EXEC% set winrm/config/service @{AllowUnencrypted="true"}
  %WINRM_EXEC% set winrm/config/service/auth @{Basic="true"}
  EOF
  cd /tmp/win7-custom-image
  vagrant up
  ```

1. After the machine boots up, set the network location to either **Home** or
   **Work**.

1. Manually update Virtualbox Guest Additions, if needed.

1. Run the `winrm.bat` script as administrator.

1. At this time the [up command](http://docs.vagrantup.com/v2/cli/up.html) will
   be probably verifying if the guest booted properly. Since you just
   configured **WinRM**, the command should terminate successfully.

1. Install the [Intel USB 3.0 drivers](https://downloadcenter.intel.com/download/21129/USB-3-0-Driver-Intel-USB-3-0-eXtensible-Host-Controller-Driver-for-Intel-7-Series-C216-Chipset-Family) (if needed).

1. Reboot your Windows virtual machine.

  ```shell
  vagrant reload
  ```

#### Preparation

This portion contains a bunch of things I manually added for my benefit -- none
of this is strictly necessary.

1. Run the following script (as administrator) to configure
   [chocolatey](https://forge.puppetlabs.com/rismoney/chocolatey) and
   [puppet](https://docs.vagrantup.com/v2/provisioning/puppet_apply.html) on
   the guest machine:

  ```text
  @echo off
  @powershell -NoProfile -ExecutionPolicy unrestricted -Command "iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))" && SET PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin
  choco install -y puppet
  ```

1. In a new terminal window, use chocolatey to install a few basic windows
   essentials (as administrator)

  ```text
  puppet module install chocolatey-chocolatey
  choco install -y calibre
  choco install -y kindle
  choco install -y dotnet4.5
  choco install -y adobedigitaleditions
  choco install -y python2
  choco install -y garmin-express
  choco install -y pip
  ```

  _There appears to be an error installing `pip`, which coincidentally also
  seems fine. I have no idea but this seems to work so I'm going to leave this
  be. Computers._

1. In a *new* terminal window as administrator

  ```text
  easy_install http://www.voidspace.org.uk/python/pycrypto-2.6.1/pycrypto-2.6.1.win32-py2.7.exe
  ```

1. Ebook management tools

  - Download and extract the latest version of
  [DeDRM_tools](https://github.com/apprenticeharper/DeDRM_tools/releases)

  - Drag and move the `DeDRM_Windows_Application/DeDRM_App` directory over to
  under the `Documents` directory

  - Open the `DeDRM_App` folder you've just dragged, and make a short-cut of
  the `DeDRM_Drop_Target.bat` file (right-click/Create Shortcut). Drag the
  shortcut file onto your Desktop.

1. Reboot your Windows virtual machine.

  ```shell
  vagrant reload
  ```

#### Configuration

1. Start up Calibre and make sure that the library points to the corresponding
   library directory in Dropbox

#### Packaging

```shell
vagrant halt
vagrant package --output "win7-ie11-custom.box" --Vagrantfile Vagrantfile
vagrant box add win7-ie11-custom.box --name "win7-ie11-custom"
vagrant destroy
cd /tmp
rm -rf /tmp/win7-custom-image
```

And that's it, you're all set!


## Day to day usage

After creating the initial customized image, bringing up different instances of your image is as trivial as:

```bash
vagrant up
```

Use the [Vagrantfile](Vagrantfile) in this repo as a starting point.


## What's with all the manual commands?!

It turned out that automating some portions of the package installations became
downright impossible! Take Adobe Digital Editions, for example. The installer
that comes with it attempts to install a Norton something or the other utility,
which does not work in _silent_ mode. This results in a chocolatey
installation that fails. I considered attempting to fix this upstream but
the time sink was just not worth it for me. It was easier for me to
install all that manually and document it.


## References

- The [gist](https://gist.github.com/andreptb/57e388df5e881937e62a) by
[andreptb](https://github.com/andreptb) that started this all!

- Vagrant [box urls](https://www.bram.us/2014/09/24/modern-ie-vagrant-boxes)
for other Windows operating systems
