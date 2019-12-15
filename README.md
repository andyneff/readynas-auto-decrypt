
An attempt to add auto decrypt external USB drives, similar to internal drives

**TODO**: Explain

**TODO**: Instructions

# How internal auto decrypt works

Magic, as far as I can tell. There is a file called `/etc/crypttab` that it supposed to describe how to decrypt the (internal) file systems on boot. However the password field says "search", which is not a feature of `/etc/crypttab`, so I am guessing they replaced whatever crypt executable that reads that file with their own custom version, and thus _magic_.

I gather "search" looks at the mounted USB devices for encryption keys. Since I don't know what they used, I just search "/media/*/data.key"

# Major steps in how this works

1. Waits for readynasd to add the initial database entry to `usb_storage`. If I don't wait until after this happens, the database is revereted later... "_magic_"
2. 