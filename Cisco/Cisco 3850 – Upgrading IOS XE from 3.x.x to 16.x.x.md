# Cisco 3850 – Upgrading IOS XE from 3.x.x to 16.x.x

First, we will need to copy the new software to the flash on the switch. This can be done with either copying from a USB drive or TFTP server. I am using TFTP since I don’t have direct access to the switch.


copy tftp: flash:


We can check the flash to verify that the new software is present

show flash:


# NOTE: You can increase the TFTP block size to increase the transfer speed.

To increase the block size enter the command below in configure terminal

ip tftp blocksize 8192show flash:
Second, we need to Regenerate RSA crypto keys before the upgrade.

crypto key generate rsa general-keys modulus 1024
Now we are ready to upgrade the switch. Before we run the upgrade command, let’s verify how many switches are in the stack:
show switch


software install file flash:/ cat3k_caa-universalk9.16.06.05.SPA.bin switch 1-4 verbose new force


If you have 3 switches for example change the switch 1-4 to switch 1-3

After, we run the command we will be prompted with a couple of questions:

Do you want to proceed with reload? [yes/no]:yes

System configuration has been modified. Save? [yes/no]:yes

The switch will reboot, and the upgrade begins, it will take approximately 15-20 mins

– After the switch comes back up, we can confirm that all switches are upgraded

show version

Now we can run the following command to clean up the old packages from the flash and free up some space

request platform software package clean switch all file flash:

