# Fortigate VM Setup - 20260520

## Getting WiFi and adding LAN bridge on mini PC

To start off my home server series, I needed to first get something I could use as a server.<br>
Due to budget restraints, to went with a used HP ProDesk mini pc. Intel i5, 8gb of ram, and 256gb of memory.<br>
As of writing this, the current plan for this server will be as follows:<br>
→ Create a web server<br>
→ Create a WAF with Modsecurity and Nginx<br>
→ Host webpages and Docker containers<br>
→ Create a log collector that will be forwarded to QRadar<br>
→ Have Fortigate VM handle network security

A few days ago I set up Ubuntu Server on my device as well as get WiFi and a static IP configured to it.

To configure the static IP, I needed to edit the netplan file found in the etc folder.<br>
`sudo nano /etc/netplan/00-installer-config.yaml`<br>

![alt text](<images/fortigate-setup/wifi.png>)

I deleted everything after "renderer" and added my wifi. Under "wifis" is the wifi usb device that came with my mini pc. I set the dhcp4 to false to prevent it from getting a new IP each time the server would turn back on. The addresses below is the local IP address of my choice with subnet (192.168.x.x/24).<br>
Below that, "routes", is set to default and the "via" part is my gateway IP address I took from my main PC.<br>
The "nameservers" is Google's IP.<br>
Lastly, the "access-points" is my WiFi router. On top is the name of the router, then the password below is well, the password for it.

Under the WiFi part, I then added my ethernet. The static ip here will be different from our WiFi static ip. Some of the additional fields are:<br>
→ eno1 is my ethernet name. You can check your's from `ip a`. We have it set to no ip (just a raw cable port) because we're letting the bridge handle it.<br>
→ br0 is the bridge that owns the cable port and gets a static ip.<br>
→ stp: false and forward-delay:0 just stops any wait time before forwarding traffic. STP protocol is just for preventing loops in big networks.

After that was all typed, I applied the new plan with `sudo netplan apply`.<br>
If you make some mistakes and need to change the plan, there's a good chance you'll need to clear the old dhcp first before the new one takes effect.<br>
You can do this with `sudo ip addr flush dev <name of wifi card/usb>`. Then apply again.<br>
Now you'll be able to ssh into ther server.

Note: When connecting your WiFi for the first time, be sure the WiFi usb device driver is installed on your server first or else you won't have any internet to your server.

Great, now that that's out of the way, let's get back to installing Fortigate VM.<br>
Quick thing about Fortigate VM, it's a free Fortigate firewall, but it's extremly limited with what you can do with it. For my home lab though, it's more than enough.

## Creating a virtual LAN network -- Bridged mode

To get started with installing Fortigate VM, we need to install a VM engine, in this case Qemu for Ubuntu. Along with it, we'll install a management layer for KVM for easy commands to control the VM, and install another command that lets us create VMs from the command line since our server is command line only.<br>
`sudo apt install -y qemu-system-x86 libvirt-daemon-system libvirt-clients virtinst`<br>
→ qemu is the VM engine.<br>
→ libvirt is the management layer.<br>
→ virtinst is to use the command line.<br>
We'll then want to enable libvirtd so it auto starts whenever the system reboots.<br>
`sudo systemctl enable --now libvirtd`

And since my current user isn't root, let's add it to the libvirt group to avoid typing sudo all the time.<br>
```
sudo usermod -aG libvirt,kvm $USER
newgrp libvirt
```
To show a list of VMs currently created:<br>
`virsh list --all`<br>
But it should be empty at the moment.<br>

First, make sure you have a LAN cable connected from your server to your router since we're doing this in bridged mode. Then, we'll need to create a virtual LAN. No internet connection goes through the LAN, and bridging allows us to connect to our physical ethernet port directly to the vm, making Fortigate's WAN port think it's physically plugged into the router. NAT will fail us here since the server was recieving the internet traffic first, translating, then passed it to Fortigate. You can access the GUI via NAT after jumping through some hoops, but this method is just much more easier and stable.

We'll make a config file for the LAN.<br>
`nano ~/vnet-lan.xml`<br>
![alt text](<images/fortigate-setup/vnet-lan.png>)<br>
→ vnet-lan is our network name.<br>
→ virbr-lan is the virtual switch name.<br>
→ The ip is what ubuntu will have inside the internal network. You can choose this.

Next run the following to register and start the network:<br>
```
virsh net-define ~/vnet-lan.xml
virsh net-start vnet-lan
virsh net-autostart vnet-lan
```

Then check if it's running with `virsh net-list --all`. If you see it there as active, you're good.<br>
![alt text](<images/fortigate-setup/vnet-active.png>)


## Getting Fortigate VM package

To use Fortigate we require an account.<br>
There's a few things we need to consider when future testing since we're going the free route.<br>
・ If you choose to go the trial version first, Fortigate VM trial only unlocks all features for 30 days free. After that, the VM will still boot and run, but it will be heavily restricted, because it's unlicenced.<br>
・ This also means that the live threat update feed database that is auto updated from Forti servers will end. If we still want to keep up with security, we'll need to manually import threat intelligence files from free public blocklists.<br>

Note: I am not going the trial version route first in this tutorial, so we'll need to adjust our settings accordingly to use the free tier.

Steps on the website:<br>
→ Create an account at support.fortinet.com.<br>
→ Go to `https://support.fortinet.com/Download/VMImages.aspx`<br>
→ Select KVM as the platform on the left-hand side.<br>
→ Download `FGT_VM64_KVM-v8.0.0.F-build0167-FORTINET.out.kvm.zip`.<br>
FGT is the product we want, our hardware is x84-64, KVM is our hypervisor, .kvm.zip contains the qcow2 disk image we need.<br>
The verison you download may vary depending on what's currently released.<br>
![alt text](<images/fortigate-setup/forti-zip.png>)

Open a new command prompt and scp file transfer the zip to the server. scp is a secure file transfer over ssh.<br>
`scp C:\Users\<username>\Downloads\FGT_VM64_KVM-v8.0.0.F-build0167-FORTINET.out.kvm.zip <server-username>@<server-ip>:~/`

Back on the server, uzip the file.<br>
Install unzip first if you don't have it `sudo apt install -y unzip`<br>
```
cd ~
unzip FGT_VM64_KVM-v8.0.0.F-build0167-FORTINET.out.kvm.zip
ls
```

## Deploying Fortigate VM

Now that we have our file on the server, time to connect it to a vm.<br>
To create our vm:<br>
First copy the unzipped .qcow file to the libvirt images folder (we changed the copied file's name from fortio to fortigate).<br>
`sudo cp fortios.qcow2 /var/lib/libvirt/images/fortigate.qcow2`<br>
```
sudo virt-install \
  --name fortigate \
  --ram 2048 \
  --vcpus 1 \
  --os-variant generic \
  --disk path=/var/lib/libvirt/images/fortigate.qcow2,format=qcow2,bus=virtio \
  --network bridge=br0,model=virtio \
  --network network=vnet-lan,model=virtio \
  --import \
  --noautoconsole
```
→ name, ram allocation, and vcpu are self-explanatory. You could add more cpu and memory, but to stay within the free tier constraints, this is the best we can do.<br>
→ disk path is the virtual hard drive we just copied over.<br>
→ The network bridge is us plugging directly into Fortigate's first port, aka our Fortigate WAN port. Fortigate will then get an ip from the router dhcp like any other device.<br>
→ Then network network plugs Fortigate's second port into the isolated network, aka our Fortigate LAN port.<br>
→ import just means to use an existing disk image instead of creaking one from scratch.<br>
→ Then noautoconsole prevents auto opening a console window since we can't see it anyway. We're working on headless, aka no GUI.

Now if we use the command from earlier, `virsh list --all` to check what vms are running, we should see our Fortigate.<br>
![alt text](<images/fortigate-setup/fortigatevm.png>)

Now to connect to the Fortigate console.<br>
`sudo virsh console fortigate`

Note: If you shutdown your server, the Fortigate VM will not auto-start on reboot like the other vms. We can set it up to do so, but as it's not that important (since it just gives us access to the console), we'll just remember to start it again each time. First `sudo virsh start fortigate` then the command above.

Now that we're connected, we're typing directly into Fortigate's terminal.<br>
From here click enter until we get to the login prompt.<br>
![alt text](<images/fortigate-setup/forti-login.png>)

Username is admin and when you click enter you'll be told to create a new password.<br>
Also, to exit the Fortigate VM to get back to your server, type `Ctrl ]`.

After that, we need to give the Fortigate LAN port a static ip address.<br>
```
config system interface
  edit port2
    set mode static
    set ip 192.168.x.x 255.255.255.0
    set allowaccess https ssh ping
  next
end
```
→ port 2 is our LAN. port 1 is our WAN, aka vnet-wan facing the internet. Management is done on the LAN for security reasons because we don't want that exposed to the internet.<br>
→ Make sure your LAN is on a different subnet from the WAN. Either change the third octlet or go with a 10.x.x.x or 172.x.x.x ip.

Now make sure that port 1, our WAN, got an ip from our router.
`get system interface physical`<br>
![alt text](<images/fortigate-setup/forti-router.png>)

We should see an 192.168.x.x ip assigned by our router's dhcp. If you don't see one, you can force it with the command:<br>
```
config system interface
  edit port1
    set mode dhcp
  next
end
```

Great! Now that our bridge is set up, we should be able to log into the Fortigate GUI from our main computer. Go to your browser and using the ip address that showed up on port 1 from `get system internal physical` search for the page `https://192.168.x.x/login`. (We will set this ip to static later, but first make sure it's working.)<br>
You should get a warning for insecure webpage, don't worry that's normal, but just bypass that and you'll get the Fortigate GUI login page.<br>
![alt text](<images/fortigate-setup/login-gui.png>)
![alt text](<images/fortigate-setup/licence.png>)

Next we need to get our Evaluation license. Select Evaluation licence as the Activation type.<br>
You'll notice the restrictions it has and why we made adjustments to our vm settings earlier.<br>
Enter your Fortigate account email and password, and it'll reboot the system.<br>
Let's see if it works!

Note: Fortigate VM only allows for 1 evaluation licence at a time. If you already have a licence activated, you'll need to remove it from your account first. This is important to know if you decided to delete and reinstall the vm later.

![alt text](<images/fortigate-setup/dashboard.png>)

Nice! We got the dashboard!

Note: If you are getting kicked out of the dashboard and sent back/redirected to the login page seconds after logging in, this is a bug that also happened to me originally. You can get around this by downgrading to v7.6.6 instead. You can download it from the same location as v8.0.0, just choose a different version from the lefthand side. (After a lot of troubleshooting I resorted to using v7.6.6 as well.)

Now let's get that static ip set up now for the WAN. You can set the ip as you choose for port 1. You can use the same one that was first set from the DHCP if you want.<br>
```
config system interface
  edit port1
    set mode static
    set ip 192.168.x.x 255.255.255.0
    set allowaccess ping https
  next
end
```
→ If we don't allow https here, we won't have access to the GUI.<br>
→ Allowing ping is mostly for troubleshooting purposes.

After that we need to set up the default gateway too so Fortigate knows how to reach the internet.<br>
```
config router static
  edit 1
    set gateway 192.168.0.1
    set device port1
  next
end
```

And that's all to it!<br>
Now that we have Fortigate up and running on our server, we can start messing around and tweaking with it.