# Web Server Setup - 20260521

The goal of ths project is to set up a web server with ModSecurity and nginx to host my personal website (and other future projects). We'll also be connecting the Fortigate we set up previously. We will be running this web server on a Docker container as well.

## Securing SSH access

First let's take care of securing port 22/ssh before we start accessing the internet.<br>
Open up `sudo nano /etc/ssh/sshd_config` and we'll want to change or add these lines:<br>
・ `PermitRootLogin no` → cannot root login at all<br>
・ `AllowUsers <username>` → only allow your user<br>
・ `PasswordAuthentication no` → disabling because we'll set up key authentication later<br>
・ `Port 2222` → changing the default port just to avoid heavy probbing<br>
・ `PermitEmptyPasswords no` → cannot have empty passwords<br>
・ `MaxAuthTries 3` → limit how many attempts

We're focusing on security here, so the above is just the bare minimum.

## Setting up a Host-level UFW (Uncomplicated Firewall)

In a privious setup, we installed Fortigate on our server, but Fortigate is only protecting the network boundary. UFW will protect the individual host. UFW is Ubuntu's built-in firewall, so lets take advantage of it for extra security. Basically if another device on our network is compromised, the UFW will add some security to the server where Fortigate can't.<br>
If you don't already have it installed, do `sudo apt install ufw -y` including -y to auto say yes to everything.<br>
And to make sure we don't get kicked out of our ssh tunnel before messing with it, make sure to allow it with `sudo ufw allow ssh`. Later we'll change our ssh port for more security.

Now let's add some other rules:<br>
・ `sudo ufw default deny incoming` → deny all incoming traffic<br>
・ `sudo ufw default allow outgoing` → allow all outgoing traffic<br>
・ `sudo ufw allow 80/tcp` → allow for web server<br>
・ `sudo ufw allow 443/tcp` → also needed for web server<br>
Then enable with `sudo ufw enable` and verify your rules with `sudo ufw status verbose`. verbose here is for more details.

## Installing Docker

We'll download Docker from their official repo.<br>
First let's get the prerequisites, which you might have already. But just in case.<br>
`sudo apt install ca-certificates curl gnupg lsb-release -y`

Next Docker's official GPG key to verify downloads are legitimate.<br>
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
And then we want to add the Docker repository.<br>
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
And finally installing Docker itself.<br>
```
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Once that's all done downloading, we'll add our user to the docker group so we don't have to sudo every command.<br>
```
sudo usermod -aG docker <username>
newgrp docker
```

One thing to note though, we need to edit Docker security to prevent them from being reachable outside the host firewall. UFW won't protect them.<br>
So open up `sudo nano /etc/docker/daemon.json` and add the following:<br>
```
{
  "iptables": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userns-remap": "default"
}
```
![alt text](<images/webserver-setup/docker-settings.png>)

→ "iptables": false stops Docker from bypassing our UFW.<br>
→ log-driver settings prevents logs from eating all our disk space, so we've limited the json file sizes.<br>
→ userns-rmap runs containers as an unprivileged user. This means if a container does happen to get compromised, it won't reach root on the host.

Great, now