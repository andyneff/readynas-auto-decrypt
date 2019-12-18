TODO:

- Add script that will encrypt the drive for you.
- Add a key for each drive, and swap it out.

Add auto decrypt external USB drives on insert, for ReadyNAS

Un-encrypted devices show up with a `USB_` prefixed device, just like any other USB device. If you plug in the encrypted device before the decryption key, it will wait 10 minutes for you to plug it in, just like on boot (but I had to make the message myself, so it's limited to one line)

# Installation

1. Clone this repo in a drive on the nas (or copy it over your own way)
1. ssh into your Nas as root, you may need to enable ssh
1. Run `./install` on the nas, as root. This well install all the files needed
  - Add a udev rule that will call the usb-decrypt service
  - Add the decrypt-usb service template that will call the `decrypt_usb` script
  - Install the decrypt_usb scripts in `/opt/usb-auto-decrypt`
  - Sync all the files with the running daemons, so you are ready to go

Now all you need to do is insert your usb key with the decryption key in one USB slot, then plug in your encrypted drive in the other, and it will mount and decrypt.

# How internal auto decrypt works

~~Magic, as far as I can tell. There is a file called `/etc/crypttab` that it supposed to describe how to decrypt the (internal) file systems on boot. However the password field says "search", which is not a feature of `/etc/crypttab`, so I am guessing they replaced whatever crypt executable that reads that file with their own custom version, and thus _magic_.~~

It appears the `/etc/crypttab` file isn't used by the readynas services. Instead, `/run/systemd/generator/systemd-cryptsetup\@namehere\\x2d0.service` runs on boot

```bash
/lib/systemd/systemd-cryptsetup attach 'namehere-0' '/dev/md/namehere-0' '/run/systemd/cryptsetup/namehere.key' ''
/sbin/btrfs device scan '/dev/mapper/namehere-0'
```

Now, there is another service `/run/systemd/generator/namehere-key.service`, that runs `/usr/bin/rnutil search_for_key 'namehere' '/run/systemd/cryptsetup/namehere.key' 600`. This is the the service that looks for the `namehere.key` file (in the root of the filesystem?) on each usb device.

I would use that same `rnutil` call, however it does not update the LCD screen, so I use my own version of this. For example, if you are trying to mount `USB_HDD_4`, it will look for `USB_HDD_4.key`

# How does this works

1. Find a decryption key, searches `/media/*/{namehere}.key`.
2. Decrypt and mount partition
3. Continues to run in the background until the drive is unmounted
  - Update database every time the usb drive is removed. This was only observed to happen any time any usb drive is removed. I think what is happening is readynas is "refreshing" the database every time. From it's point of view, `/dev/sdx1` has no recognized filesystem, and is not mounted, so it updates as that, which is why it gets cleared
  - It is unclear why when I make my own entry (for example `UUSB_HDD_1`), it is actually unmounted. Again, I assume it is some type of cleanup routine that does not understand my entries.
  -I have literally tracked EVERY file on disk. The only thing left to explain these discrepancies is the state of running daemons, which I can't fix.

As long as the databases are updated with the entries, it's pretty much like a normal volume, in the web UI.

# Bugs

All bugs have been fixed on my NAS. If you see any, please report and I'll see what I can guess :)

# Troubleshooting

In these example, `/dev/sdg1` is the hard drive partition. Yours will be different, and may not even have the 1 at the end, depending on how you formatted your drive.

* How to manually run script: `/opt/usb-auto-decrypt/decrypt_usb /dev/sdg1`. The only intended output is when the database is updated, so that should help tell you when you should see it in the UI. (add `set -xv` to turn on verbose mode for any bash script)
* How to manually run service: `systemctl start usb-decrypt@/dev/sdg1`, for example with `sdg1` being the device name
  * `systemctl status usb-decrypt@/dev/sdg1` should show the last 5 lines or so of output from the script, to help debug
* How to test the udev rule: `udevadm test --action=add $(udevadm info /dev/sdg1 -q path)`
  * Towards the end, you should see `SYSTEMD_WANTS=usb-decrypt@/dev/sdg1.service`. This is good. If you don't, your device/setup may be different from mine and the rule is not picking it up. Try adjusting the rule.
