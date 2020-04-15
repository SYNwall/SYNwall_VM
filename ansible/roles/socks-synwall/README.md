Role Name
=========

This role installs and configures Dante socks server

Role Variables
--------------

First we set where to download the Dante sources, to the `url` we append the name of the archive with the version in it `dante-{{ dante.version }}.tar.gz`.
The `checksum` and `checksum_alg` are there to control the downloaded version.

`download_dir` tells the role where to download Dante.

In the `conf` section you can set some values for the Dante socks configuration file.

    dante:
      url: "https://www.inet.no/dante/files"
      checksum: "4c97cff23e5c9b00ca1ec8a95ab22972813921d7fbf60fc453e3e06382fc38a7"
      checksum_alg: "sha256"
      version: "1.4.2"
      download_dir: "/opt"
      conf:
        path: "/etc/dante.conf"
        log_path: "/var/log/danted.log"
        interface: "enp0s3"

This part of the configuration is to open the encrypted partition, sync this with your `kickstart.cfg` file

    luks:
      volgroup: "modprobe"
      logicalvol: "modprobe"
      name: "modprobe"
      mount_point: "/etc/modprobe.d"
      fstype: "ext4"

Here is the configuration for the SYNgate module. You can define where to download it from, where to download it on the system and where to install the compiled module.

The parameters are mandatory, do not delete them, instead set them to 0 if you don't need the feature. All the parameters are CSV lists.

    syngate:
      module_dir: "/lib/modules/{{ ansible_kernel }}/extra/"
      repo: "git@wecode.sorint.it:cpizzi/SYNwall.git"
      path: "/opt/SYNwall/"
      params:
        dstnet_list: "0.0.0.0"
        # Pre-Shared Key used for the OneTimePassword
        psk_list: "qazwsxedcrfvtgbyhnujmikolpqazwsx"
        # Time precision parameter
        precision_list: 1
        # Enable IP Spoofing protection
        enable_antispoof_list: 0
        # Enable OTP for UDP protocol list
        enable_udp_list: 0

Example Playbook
----------------

    - name: Install Dante socks
      hosts: ...
      become: true
      connection: local
      roles:
        - socks-synwall

License
-------

GPL-3.0

Author Information
------------------

[Sorint.Lab](https://www.sorint.it)
