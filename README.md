
Add auto decrypt external USB drives on insert, for ReadyNAS

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

1. Waits for readynasd to add the initial database entry to `usb_storage`. If I don't wait until after this happens, the database is revereted later... "_magic_" and it will not work. We have to let the ReadyNAS fail to mount first, then we can come in and fix it
2. Find a decryption key, searches `/media/*/data.key`. _Better_ ideas welcome, if this doesn't work for you
3. Decrypt and mount partition
4. Update database

After this, it's pretty much like a normal volume, in the web UI.