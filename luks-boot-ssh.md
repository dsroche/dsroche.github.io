# LUKS full-disk decryption on boot using SSH

This short note says how to decrypt a LUKS-encrypted volume on Linux
remotely using SSH, at boot time.

Specifically, the idea is to get `dropbear` (a small SSH server) running
in your `initramfs` (the initial mini-Linux system that boots before your
actual OS boots). You set up dropbear to look at a specific
`authorized_keys` file and tell the initramfs how to set up its IP
address and such. Then you can SSH from another machine into this one
just to type in the decryption password.

## Prerequisites

Your disk should have an **un**encrypted `/boot` partition. (There is
probably also an unencrypted `EFI` partition at `/boot/efi`.)
Make sure the encrypted partition(s) are listed in `/etc/crypttab`.

I have tested this on Debian 12 (bookworm) and sid. Fairly confident
it would work on Ubuntu flavors as well.

Make sure that you can already ssh normally from the *client* machine to
the *target* using SSH public/private keys.
(The target is the one with full-disk encryption that you
want to boot remotely from the client.)

## Setup steps

Do this on the *target*.

1)  Install packages

    ```
    sudo apt install dropbear-initramfs console-setup
    ```

2)  Take note of SSH host key ED25519 fingerprint from install above.

    Or, you can get the fingerprint later by running

    ```
    sudo dropbearkey -y -f /etc/dropbear/initramfs/dropbear_ed25519_host_key
    ```

    (This will be needed by the *client* machine.)

3)  Edit dropbear config

    ```
    sudo -e /etc/dropbear/initramfs/dropbear.conf
    ```

    You want to change `DROPBEAR_OPTIONS` to customize how the ssh
    server will run. Try this line:

    ```
    DROPBEAR_OPTIONS="-FEsjk -p 22 -c /usr/bin/cryptroot-unlock"
    ```

    Explanation:

    *   `-F`: run in foreground
    *   `-E`: log to stderr
    *   `-s`: disable password login
    *   `-jk`: disable all port forwarding
    *   `-p 22`: change if you want a non-standard port
    *   `-c /usr/bin/cryptroot-unlock`: run this command (only) instead
        of a shell

4)  Edit IP address config

    Run `ip a` if you're not sure about your network device name and IP
    address info.

    ```
    sudo -e /etc/initramfs-tools/initramfs.conf
    ```

    Add one line to the bottom of this file:

    *   For **static** IP address configuration:

        ```
        IP=MYIP::GATEWAY:NETMASK:HOSTNAME:DEVICE
        # for example
        IP=10.60.149.224::10.60.149.1:255.255.255.0:lnx1054670govt:enp1s0
        ```

    *   For **DHCP** automatic configuration:

        ```
        IP=::::HOSTNAME:DEVICE:dhcp
        # for example
        IP=::::lnx1082409govt:enp1s0:dhcp
        ```

5)  Add network device module

    (Sometimes will work without doing this...)

    First note the kernel module of your Ethernet controller by running:

    ```
    sudo lspci -kd ::0200
    ```

    For example, `atlantis` or `e1000e`. Then add this line at the end
    of the config file for kernel modules to load in the initramfs:

    ```
    sudo -e /etc/initramfs-tools/modules
    ```

6)  Add authorized keys

    This line will add *all* authorized keys to SSH (as current user) to
    allow them to SSH at boot time for disk decryption:

    ```
    cat ~/.ssh/authorized_keys | sudo tee /etc/dropbear/initramfs/authorized_keys
    ```

7)  Rebuild the initramfs

    ```
    sudo update-initramfs -u -k all
    ```

8)  Reboot and cross your fingers!

    At reboot, after GRUB you should see a prompt for the decryption
    password AND output from dropbear showing the IP address
    configuration.

## From the client machine

After doing all the setup above on the target, add an entry to
`~/.ssh/config` like this:

```
host TARGETBOOT
    hostname TARGET.ACTUAL.HOSTNAME
    hostkeyalias TARGETBOOT
    user root
```

You can make `TARGETBOOT` be anything you want. The reason for this is
to not confuse ssh with multiple host keys, since the host keys for
dropbear at boot are different from those in your actual OS after boot.

Now, once the target machine is rebooted and waiting for a password, try

```
ssh TARGETBOOT
```

If all goes well, you should be prompted for a passphrase to decrypt the
chosen partition(s) on the target machine. Then, on the client, you will
see a confirmation that you entered the correct password, and the SSH
session immediately ends. In a moment (once the target machine OS comes
up), you should be able to SSH normally into it.
