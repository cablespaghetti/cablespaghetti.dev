---
layout: post
title:  "DiskCryptor full disk encryption"
description: "A few months ago I was tasked with encrypting the laptops of a few of our consultants who travel a fair bit. I was going to go with Microsoft BitLocker, but as we use Windows 7 Professional we're out of luck. I also looked at using TrueCrypt, but there are a lot of question marks around it since it was abandoned by its developers..."
date:   2015-02-28 19:12
tags: encryption
redirect_from:
  - /2015/02/28/diskcryptor.html 
---
A few months ago I was tasked with encrypting the laptops of a few of our consultants who travel a fair bit. I was going to go with Microsoft BitLocker, but as we use Windows 7 Professional we're out of luck. I also looked at using TrueCrypt, but there are a lot of question marks around it since it was [abandoned by its developers](http://truecrypt.sourceforge.net/) and its successor [VeraCrypt](https://veracrypt.codeplex.com/) wasn't around at the time I was looking. There are also a lot of enterprise paid options, but for our usage the free options are perfectly fit for purpose.

DiskCryptor is very easy to set up on an existing Windows installation, and because it uses the hardware AES encryption support built into most modern processors there is no perceivable effect on system performance. However I did find that the documentation, especially around the forgotten password reset process, was a bit lacking. So here is the documentation I wrote for our internal wiki (with some small changes), which covers the encryption process and what to do if you/one of your users forgets their password.

**Table of Contents**

* [Encrypting a machine](#encrypting-a-machine)  
* [Recovery Process when the password has been lost](#recovery-process-when-the-password-has-been-lost)
* [Changing the password after initial encryption](#changing-the-password-after-initial-encryption)

# **Encrypting a machine**

1. Go to [the DiskCryptor website](http://diskcryptor.net) and get the latest DiskCryptor installer (we used 1.1.846.118), then reboot the machine when prompted.
2. After you have rebooted the machine, run DiskCryptor from the Start Menu.
3. Select C: from the list (assuming this is your boot drive), and click on Encrypt.
4. Continue with the default Encryption Settings and Boot Settings; these will be very secure and the best performing for most fairly modern machines.
5. Pick a Volume Password and store this somewhere very safe such as a [Lastpass](https://lastpass.com) Vault.
6. The system will then encrypt your hard disk. It takes about an hour for a 256GB SSD. ***DO NOT CANCEL THIS OR TURN OFF YOUR MACHINE!***
7. When the encryption process has finished, select C: from the list again and click on Tools > Backup Header. Save this somewhere very safe ***off of your hard disk*** such as Dropbox, Google Drive or a backed up file server.
8. You can now reboot the machine. You should be prompted for the password you picked in step 5.

*DiskCryptor uses a US keyboard layout during boot (not when you set the password in Windows). So some special characters might be in different places.*

# **Recovery Process when the password has been lost**

To understand what you're doing you need to understand a bit more about how DiskCryptor works when it encrypts your disk.

When you click Encrypt on a disk it generates a unique encryption key which is used along with the AES cipher to encrypt and decrypt your data. This encryption key is stored in an encrypted form in the boot sector of your hard disk. When you turn on your machine the password you set is used to decrypt the encrypted encryption key. DiskCryptor can then access the encrypted data and boot Windows. If you change the password, it doesn't change the encryption key; it just decrypts it with the old password and encrypts it with the new one you've set.

So when we backed up the header after first encrypting the machine, we backed up the encryption key when it was encrypted with the initial password which we then stored somewhere super-safe. To recover access to the disk, we just have to restore this old header, reboot the machine and boot it up with the old password.

Because Windows won't boot to run the DiskCryptor software, you need the DiskCryptor Recovery Environment CD, which boots up into a Windows 7 environment with DiskCryptor installed. You can build your own with [this guide](https://diskcryptor.net/wiki/LiveCD) or use this admittedly slightly dodgy looking [pre-built iso image](https://diskcryptor.net/forum/index.php?topic=4899.0).

1. Boot the machine to the CD.
2. On another machine, download the header backup and put it on a network share or USB drive. If you used a USB drive, skip the next step.
3. Open Explorer from the Desktop, Click on Network and Turn on network discovery and file sharing.
![Turn on network discovery and sharing](/media/diskcryptor1.png)
4. Open DiskCryptor from the Desktop, select C:, then click on Tools > Restore Header.
![Open header backup file](/media/diskcryptor2.png)
5. Type the URI for your network share or USB drive in the File name box e.g. \\\server\users\cablespaghetti, press Enter then give it your domain username (in the domain\username format) and password.
![Grab the file from the network share](/media/diskcryptor3.png)
6. You should then be able to select the Header Backup file.
7. Reboot the machine and you should be able to unlock the encryption with the old password which I hope you still have saved somewhere super-safe.

# **Changing the password after initial encryption**

1. Run DiskCryptor from the Start Menu.
2. Select C: from the list and click on Tools > Change Password.
3. Give it your old password and set a new one.
4. Click OK.

*At this point the original password should still be kept somewhere super-safe. This old password is used along with the Header Backup to recover a machine when the password has been forgotten.*

