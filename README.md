# Packer configuration for the socks server

This configuration helps you to install the socks server with SYNwall
Packer is configured to create a virtual machine based on the CentOS/RedHat distribution.
The VM will have the Dante socks server and SYNwall module installed and running on it.

## Packer configuration

The provided configuration is a template you can work on, please read the [Packer documentation](https://packer.io/docs)

Here we explain some of the main configuration options.

### packer_config.json

This is the main configuration of Packer. This is in JSON format.

This configuration is a template, some values are taken from the variables file, anything in double braces `{{}}` is interpreted by the [Packer templating engine](https://packer.io/docs/templates/engine.html). The `{{ user '' }}` values are taken from the variables file.

#### Builders

We use a VirtualBox builder, you need VB to be installed on your local system.

+ `type`: here you can we we use a VirtualBox builder to create an ISO
+ `name`: name of the builder
+ `boot_command`: commands to type when the virtual machine is first booted, here we tell the system where to find the kickstart configuration file for the OS installation/configuration
+ `boot_wait`: time to wait after booting the initial virtual machine before typing the boot_command
+ `disk_size`: the size, in megabytes, of the hard disk to create for the VM
+ `guest_os_type`: to view all available values for this run VBoxManage list ostypes
+ `headless`: change this to true if you don't want the VM window to open
+ `http_directory`: the files in this directory will be available over HTTP that will be requestable from the virtual machine
+ `iso_urls`: the ISO to download
+ `iso_checksum_type`: algorithm to be used when computing the checksum
+ `iso_checksum`: the checksum for the ISO file
+ `ssh_username`: the username to connect to SSH with
+ `ssh_timeout`: the time to wait for SSH to become available
+ `shutdown_command`: command to use to gracefully shut down the machine once all the provisioning is done
+ `virtualbox_version_file`: path within the virtual machine to upload a file that contains the VirtualBox version that was used to create the machine
+ `format`: ovf or ova
+ `hard_drive_interface`: type of controller that the primary hard drive is attached to
+ `vm_name`: this is the name of the OVF file for the new virtual machine
+ `vboxmanage`: custom VBoxManage commands to execute in order to further customize the virtual machine being created, here we set the value for the memory and the number of CPUs

#### Provisioners

We use Ansible to configure the machine image after booting.

For the Ansible configuration, please look at `./ansible/README.md`.

### variables.json

You should configure this file according to your needs. The meaning of each value is already explained above.

### kickstart.cfg

This is the method we use to automate the installation of the RedHat/CentOS system.

In the `http` folder you can find the [kickstart](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/installation_guide/ch-kickstart2) file.

This file is not a template and can'0t be modified by Packer, you have to configure it directly.

Here you should configure all you need for your system. The installation won't be graphical.

For doubts, please read the [docs](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-syntax).

We pass through the main points:

+ Layout of the keyboard

        keyboard --vckeymap=us --xlayouts='us'

+ Language of the system

        lang en_US.UTF-8

+ Perform the installation in text mode

        text

+ Network insformation

        network  --bootproto=dhcp --device=enp0s3 --ipv6=auto --activate
        network  --hostname=localhost.localdomain

+ Root password.

    You can remove the `--iscrypted` flag and write it in plaintext

    To create an encrypted password run `openssl passwd -1 “mypassword”`. Here we use `synwall` as password

        rootpw --iscrypted $1$2/zeliDs$8qd0tLsBBJtPWQITtpXhx0

+ We enable the NTP

        services --enabled="chronyd"

+ Timezone information

        timezone America/New_York --isUtc

+ Bootloader configuration. Specifies how the boot loader should be installed

        bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda

+ Partition clearing information

        clearpart --none --initlabel

+ Disk partitioning information. We use LVM to create the partitioning and an ext4 FS. Here we create a `/synwall` encrypted partition. Here we set a passphrase for this encrypted partition. If you prefer to not have the password in a file, remove the `--passphrase` option and the installation will prompt you to insert a password
  - The size of the partition shouldn't exceed the disk size you set with Packer
  - In `part` and `logvol` the `--size` is in MiB
  - In `volgroup` the `--pesize` is in KiB
  - We do not mount the encrypted filesystem at boot. This is because Packer needs to run an Ansible provisioner after reboot, and we can't unlock the partition. We do mount it and unlock it in the Ansible provisioner.

        part /boot --fstype="ext4" --ondisk=sda --size=1024
        part pv.01 --fstype="lvmpv" --ondisk=sda --size=1020
        part pv.02 --fstype="lvmpv" --ondisk=sda --size=8195
        volgroup centos --pesize=4096 pv.02
        volgroup modprobe --pesize=4096 pv.01
        logvol /  --fstype="ext4" --size=7168 --name=root --vgname=centos
        logvol swap  --fstype="swap" --size=1023 --name=swap --vgname=centos
        logvol /etc/modprobe  --fstype="ext4" --size=1016 --encrypted --name=modprobe --vgname=modprobe --passphrase=synwall --fsoptions="noauto"

+ Package section. Here we describe the software packages to be installed.

        %packages
        @^minimal
        @core
        chrony
        kexec-tools
        %end

+ Configure the kdump kernel crash dumping mechanism

        %addon com_redhat_kdump --enable --reserve-mb='auto'
        %end

+ Additional Anaconda installation options. Here we set the default password policy for the users

        %anaconda
        pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
        pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --notempty
        pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
        %end

+ Post installation script.

    Here we install Ansible & Git (needed by Packer for the local provisioner) and update the system. In `/root/ks-post.log` you can find a log of the operation performed, use it to debug eventual errors.

    Then we add the `noauto` option in the `/etc/crypttab` file, so the encrypted partition won't be automatically unlocked at boot (which prompts the user for a password). This will be changed in the Ansible provisioner.

        %post --log=/root/ks-post.log
        yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y
        sudo yum install ansible -y
        yum install git -y
        yum update -y
        yum clean all
        sed -e $LINE's/$/ noauto/' -i /etc/crypttab
        %end

+ Reboot the VM after the installation is successfully completed

        reboot

## Run Packer

To run Packer use this command `packer build -var-file=variables.json packer_config.json`

Run `packer build -h` to see all the possible options.

To debug errors you can use the `-on-error=ask`.

### Output

You'll have an ISO with Dante socks and SYNwall installed on it.

## Cross file references

Some values are configured in more than one file and should match, here we go through them:

+ Network interface

    The name of the network interface should match in the kickstart file and in the Ansible provisioner configuration, it will be later used to configure Dante socks.

+ `root` password

    It should match in the kickstart file and in the Packer variables file, we use `root` user to ssh into the VM.

+ Disk size

    The disk size in the partition configuration in the kickstart file should not exceed the size assigned to the machine in the Packer variables file.

+ LUKS partition

    The values of the volume group name, the logical volume name, the encryption password and the mount point, should match between the kickstart file and the Ansible provisioner configuration.

# License

GPL-3.0