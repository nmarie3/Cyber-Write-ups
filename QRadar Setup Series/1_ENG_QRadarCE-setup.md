# QRadar Community Edition Installation Setup - 20260412

Setting up QRadar Community Edition is kinda a pain, and can take like 2 hours to finish.<br>
This tutorial is specifically for setting it up on a virtual machine on your host computer.<br>
So let's get started.<br>
You'll need to have an IBM account and Oracle Virtual Box installed first.

Once you have your IBM account, visit https://www.ibm.com/community/101/qradar/ce/ to download your files.<br>
What each of those files do is:<br>
1. .iso -- The actual QRadar operating system + software installer. This is the main file to install QRadarCE on a vm.<br>
2. .sha256 -- A checksum file containing a unique fingerprint of the ISO. This varifies the ISO downloaded correctly and wasn't corrupted.<br>
3. .iso.sig -- A cryptographic signature from IBM proving the ISO is genuine and came from them. Proves authenticity.<br>
4. .key -- The 3-month CE license file. Re-downloaded every 3 months to renew.<br>
The .key file is especially important, but we'll touch on that later below.

![alt text](<images\install\downloads.png>)

Next open up Virtual Box and click the New icon. This is where we'll add our .iso.<br>
Give your VM a name. I went with QRadar (later changed to QRadar CE).<br>
Below that is where you want to store the machine it creates. You can keep the default or specify your own.<br>
Next is the ISO image. This is the .iso file we downloaded from IBM. Locate it and select.<br>
Then click Next. (UI may differ. If you don't see a Next button, click the next dropdown category).

![alt text](<images\install\VB1.png>)

You'll be required to set up a user name and password. Feel free to choose what you want.<br>
Then Next.

![alt text](<images\install\VB2.png>)

It's important to get this part right. QRadar has very specific requirements to run correctly.<br>
It's recommended you have 24 GB for RAM/memory, 6 CPU cores, and 250 GB of disk space.<br>
You might be able to get away with 4 cores, but if you can do 6, opt for 6.<br>
Once that's all set, click Finish and your vm should start up automatically.

![alt text](<images\install\VB3.png>)
![alt text](<images\install\VB4.png>)

Except, it might abort instead. Mine did with an error about certain files pre-existing.<br>
To solve this issue, I deleted these random four files that appeared in the VM folder where the vm was created.<br>
I'm not entirely sure what they're for or why they were created, but after I deleted them and restarted the vm again, everything was fine.<br>
It will tell you you need to remount the DVD after you delete them, so just add the .iso file again.

![alt text](<images\install\aborted.png>)
![alt text](<images\install\upload-iso.png>)

But one thing first!<br>
Before you start your QRadarCE vm, we want to go into the settings and change the adapter.<br>
Unless you're importing from your host, then NAT is probably fine. However...<br>
The goal of this lab is to send real log data from a seperate device on the network--this being my home server.<br>
The vm will automatically set as NAT unless you go in and change it, so be sure to do it now.<br>
You can this by right clicking the vm > Settings > scroll down until you find the Network.

![alt text](<images\install\bridged.png>)

Now let's start up the vm.<br>
You'll first be asked to log in. This will be `root` and the password is blank.<br>
After that, the installation will begin. This can take about an hour to complete.

![alt text](<images\install\hostlogin.png>)
![alt text](<images\install\installing.png>)

Once the initial installation completes, you'll be greeted with a very, veryyyy long ToS.<br>
To skip through it all you can press `q`. Then type in `yes` to accept.

![alt text](<images\install\bootTOS.png>)
![alt text](<images\install\bootTOS2.png>)

Some more installing will go on, give or take 5~10 mins.<br>
Until the QRadarCE Red Hat boot screen appears.<br>
Great, we're halfway there.<br>
Go ahead and tap Enter for the first selection.

![alt text](<images\install\bootscreen.png>)

The next screen will be selecting which we're installing.<br>
Choose the software install option. To make a selection, tap the space bar.<br>
Next keep it at "All-In-One" Consle. Then keep at Normal Setup.<br>
Next you'll reach a screen asking to choose time server name or ip address.<br>
You can choose to keep the preset, or you can set it too `pool.ntp.org`.<br>
pool.ntp.org is a public NTP pool that is free.

![alt text](<images\install\setup1.png>)
![alt text](<images\install\setup2.png>)
![alt text](<images\install\setup3.png>)
![alt text](<images\install\setup4.png>)

After that just go through the rest of selection location, time zone, ipv4, and enp0s3.

![alt text](<images\install\setup5.png>)
![alt text](<images\install\setup6.png>)
![alt text](<images\install\setup7.png>)
![alt text](<images\install\setup8.png>)

The next screen we'll be filling out your hostname, ip address, network mask, gateway, primary dns, secondary dns, and public ip.<br>
Pick a hostname. Something similar to what I have will work.<br>
Go back to your host machine and open the command prompt and enter `ipconfg`.<br>
This will give us our current IP, subnet, and default gateway.<br>

![alt text](<images\install\ipconfig.png>)

We can't use the exact IP as our host for our QRadar server, so just change the last part to some three-digit number under 255.<br>
Then you can copy the subnet/network mask and gateway.<br>
What's important to note here is that the first three parts/sections of the gateway and the ip address must be the same. Otherwise you'll get an error.<br>
Using 8.8.8.8 (Goggle's DNS) as our primary DNS is reliable to solve hostnames.<br>
8.8.4.4. is Google's secondary public DNS.<br>
Then for the public ip, this will be the same as the ip address you inputed earlier.

![alt text](<images\install\setup9.png>)

Next you'll be asked to set up a new admin password. This will be the password you use to log into the web console.<br>
But don't worry too much, you'll probably be asked to reset it anyway when you sign in for the first time.<br>
After that, you'll be asked to make a new root password. This password is for the vm terminal you're on.

![alt text](<images\install\setup10.png>)
![alt text](<images\install\setup11.png>)
![alt text](<images\install\installdone.png>)

Once that's all finished, congrats! You've completed the installation process!<br>
Now to see if your web console is working.<br>
Go to your browser and logon to `https://<your-qradar-ip>/console` replacing your-qradar-ip with the ip you entered during the setup.<br>
If you forget what your ip address is, you can type `ip a` into the vm terminal and find your ip (look for enp0s3 for virtual box then find inet under it.).

![alt text](<images\install\ipa.png>)

If successful, you should see a UI like this.<br>
To sign in, the login will be "admin" and the password is the password you entered earlier.<br>
If you forgot what that password is already, you can reset it from the vm terminal:<br>
Type `/opt/qradar/bin/qradar_password.sh` and this will prompt you to enter and confirm the new password.

![alt text](<images\install\qradar-login.png>)

You should be able to see the QRadar CE dashboard now!
Awesome!

![alt text](<images\install\dashboard.png>)

Now let's just touch on one last thing. Licence keys.<br>
You won't need to worry about this the first time you set up the server, but when you log into the web console, you'll notice a warning pop up saying your licence will expire on so-and-so date.<br>
Below is how to avoid having your setup expire.

![alt text](<images\install\update.png>)

Remember on the download page there were four files? One of them was a licence key, a .key file.<br>
While QRadar CE is free for as long as you want to use it, IBM updates the keys every thress months. Which means that after the three months (shorter depending when you downloaded), QRadar will stop recording logs. In order to keep using it, you will have to download the new key and import it to your server.<br>
Sounds like a pain, but this is actually pretty easy.

Instead of going through the vm terminal, we can ssh into it and import.<br>
From your host machine, go to your ①terminal/command prompt and type `ssh root@<qradar-ip-address>` replacing qradar-ip-address with the ip you gave it.<br>
Then open up another ②terminal on your host machine and past this:<br>
`scp <path-to-qradar-key/qradarkey.key> root@<qradar-ip-address>:/tmp/`<br>
Obviously replacing the <> fields with the correct info.<br>
Easist way to make sure your file path is correct is to drag and drop the file into the terminal.

![alt text](<images\install\adding-key.png>)

What scp is doing is copying a file from outside (in this case your host) into the qradar vm's tmp folder.<br>
It stands for Secure Copy Protocol--used to securely copy files between machines over ssh over a network.<br>
scp [source] [destination]

To check the file got there safely you can go back to your ssh session terminal and type:<br>
`ls /tmp/*.key`<br>
This will show all .key files in there.

![alt text](<images\install\check-key.png>)

After that, we then need to apply the new key from the ssh terminal with:<br>
`/opt/qradar/bin/qradar_license_key_upload.sh /tmp/<new-.key-file>`

And you're all set for another three months!<br>
You can also check to see your uploaded keys from the web console.<br>
From the dashboard, find the Admin panel by clicking the left-side hamburger sidebar.<br>
Then click System and License Management. Your licence should show up there!

![alt text](<images\install\sidebar.png>)
![alt text](<images\install\sysconfig.png>)
![alt text](<images\install\licencelist.png>)

And if you get a warning about your .iso being inaccessible (your vm should still boot up regardless), you can fix this issue by right clicking into your vm settings and go to Storage.<br>
You'll see a yellow triangle icon for your QRadar .iso file. Delete that file and click okay.<br>
The warning will disappear when you boot up the vm next time.

![alt text](<images\install\warning.png>)
![alt text](<images\install\750-qradar.png>)

### Installation complete!