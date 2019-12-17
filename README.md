
Add auto decrypt external USB drives on insert, for ReadyNAS

**Note** No matter what I tried, the usb device never showed up on the "System" page under "Devices", it does however show up under "Shares" and in "Backup", and can be unmounted from the "Shares" tab.

Unencrypted devices show up with a `UUSB_` prefix, "unencrypted USB drive"

# Installation

1. Clone this repo in a drive on the nas (or copy it over your own way)
1. ssh into your Nas as root, you may need to enable ssh
1. Run `./install` on the nas, as root. This well install all the files needed
  - Add a udev rule that will call the usb-decrypt service
  - Add the decrypt-usb service template that will call the `decrypt_usb` script
  - Install the decrypt_usb script in `/opt/usb-auto-decrypt`
  - Sync all the files with the running daemons, so you are ready to go

Now all you need to do is insert your usb key with the decryption key in one USB slot, then plug in your encrypted drive in the other, and it will mount and decrypt.

# How internal auto decrypt works

Magic, as far as I can tell. There is a file called `/etc/crypttab` that it supposed to describe how to decrypt the (internal) file systems on boot. However the password field says "search", which is not a feature of `/etc/crypttab`, so I am guessing they replaced whatever crypt executable that reads that file with their own custom version, and thus _magic_.

I gather "search" looks at the mounted USB devices for encryption keys. Since I don't know what they used, I just search "/media/*/data.key"

# How does this works

1. Find a decryption key, searches `/media/*/data.key`. _Better_ ideas welcome, if this doesn't work for you
2. Decrypt and mount partition
3. Update database

After this, it's pretty much like a normal volume, in the web UI.

# Bugs

1. The empty mount dir is left behind in `/media`.

# Troubleshooting

In these example, `/dev/sdg1` is the hard drive partition. Yours will be different, and may not even have the 1 at the end, depending on how you formatted your drive.

* How to manually run script: `/opt/usb-auto-decrypt/decrypt_usb /dev/sdg1`. This runs in verbose mode, so plenty of info to find errors with.
* How to manually run service: `systemctl start usb-decrypt@/dev/sdg1`, for example with `sdg1` being the device name
  * `systemctl status usb-decrypt@/dev/sdg1` should show the last 5 lines or so of output from the script, to help debug
* How to test the udev rule: `udevadm test --action=add $(udevadm info /dev/sdg1 -q path)`
  * Towards the end, you should see `SYSTEMD_WANTS=usb-decrypt@/dev/sdg1.service`. This is good. If you don't, your device/setup may be different from mine and the rule is not picking it up. Try adjusting the rule.
