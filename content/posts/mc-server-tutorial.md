+++
title = "Tutorial: Set up a Minecraft server on Linux"
date = "2024-09-26T19:45:44-06:00"
author = "Jackson Pyrah"
authorTwitter = "" #do not include @
cover = ""
description = ""
showFullContent = false
readingTime = false
hideComments = false
tags = ['gaming', 'server']
+++

Recently, I decided to start hosting a Minecraft server for me and a few of my friends. While this was mostly because...well...I wanted to play Minecraft, it also occured to me that this is an excellent project for a less experienced user to familiarize themselves with some key skills and concepts that any Linux administrator will find useful.

By the end of this tutorial, we will have:
- A Minecraft server running for the gaming enjoyment of you and your friends.
- A custom systemd service to supervise the server process.
- Appropriately configured firewall rules to allow your friends access to the server.


I will assume you already have a suitable system with any modern GNU/Linux distribution installed. For this tutorial, I will be using an Ampere Altra VPS from Oracle Cloud with AlmaLinux 9.4, but the instructions should be mostly identical for any systemd-based Linux distribution. Recommended specs for a Minecraft server depend heavily on which type of server you would like to run (vanilla, spigot, modded, etc) as well as how many players you are expecting to support, so I will not be going over this here.

You'll also need git and gcc installed for installing mcrcon.

With all that out of the way, let's jump in!

# Prerequisites

We'll start by obtaining a suitable version of Java depending on the Minecraft version we would like to host (more information on that [here](https://docs.mcserversoft.com/advanced/java-version)). In my case, I am running a modded 1.20.1 server, so I'll install Java 17 with my distribution's package manager (adjust accordingly for your distribution).

``` 
# dnf install java-17-openjdk 

```

Now we'll download and install [mcrcon](https://github.com/Tiiffi/mcrcon) according to its official installation instructions. This is a basic command-line utility that will allow us to easily manage our server from the command line via the RCON protocol. A popular alternative to using RCON as the primary management interface is to simply launch the server in a screen/tmux session that we can attach to whenever you want to run commands from the server side, but I find this a bit clunky when used in combination with systemd, as we'll get to later.

```
$ git clone https://github.com/Tiiffi/mcrcon.git
$ cd mcrcon
$ make
$ sudo make install
```

Let's prepare the ```minecraft``` system user, as well as some directories.

```
# useradd --system minecraft
# mkdir -p /opt/minecraft/{server,backup}
```

Our server will be installed into the ```/opt/minecraft/server``` directory. We also have a ```backup``` directory for world backups should we choose to utilize it.

# Installing the Minecraft server

Now it's time to download and install our Minecraft server. This will vary depending on the type of server you want to install, so I've included some links to helpful documentation for each.

- [Official vanilla server](https://www.minecraft.net/en-us/download/server)
- [Spigot](https://www.spigotmc.org/wiki/spigot-installation/)
- [Forge](https://minecraft.fandom.com/wiki/Tutorials/Setting_up_a_Minecraft_Forge_server)


Once the server .jar file has been placed in the ```/opt/minecraft/server``` directory, I like to manually start the server at least once to generate the world and populate the directory with the various configuration files you can use to manage your server.

First let's change the ownership of the ```/opt/minecraft``` directory to the minecraft user.

```
$ sudo chown -R minecraft:minecraft /opt/minecraft
```
Accept the Minecraft [EULA](https://aka.ms/MinecraftEULA) writing the following to ```eula.txt```.
```
eula=true

```
Then we can start the server.

```
# sudo -i -u minecraft
$ java -Xmx4096M -Xms1024M -jar server.jar --nogui
```

Change the ```-Xmx``` and ```-Xms``` arguments to your desired maximum and minimum RAM allocations respectively for the server, and ```server.jar``` to whatever the name of the jarfile for the server you are running is. Once the server is loaded you'll be dropped into the server console. Type ```stop``` to shutdown the server and exit back to your shell.

In the ```server.properties``` file, make sure you set the ```rcon.password``` field to a password of your choice and set ```enable-rcon``` to ```true```.

# Managing the server with systemd

Lets start by creating some scripts in the server directory to start and stop our server.

```
#!/bin/sh

### start.sh

java -Xmx4096M -Xms1024M -jar server.jar --nogui
```
Like earlier, edit the Java arguments and jarfile name appropriately.

```
#!/bin/sh

### stop.sh

mcrcon -H localhost -p rconpassword 'stop'
```
Set ```rconpassword``` to the password you set earlier in ```server.properties```.

### Note for SELinux users: ###

If you're using SELinux, you'll have to change the context of these scripts to allow systemd to execute them.

```
# semanage fcontext -a -t bin_t "/opt/minecraft/server(/.*)?"
# restorecon -R /opt/minecraft/server
```

In this case I just decided to change the context of the entire directory in case I wanted to add more/different scripts later.

Now we can create our systemd unit file in ```/etc/systemd/system/minecraft.service```.

```
[Unit]
Description=Minecraft Server

Wants=network.target
After=network.target

[Service]
User=minecraft
WorkingDirectory=/opt/minecraft/server

ProtectHome=read-only
ProtectSystem=full
PrivateDevices=no
NoNewPrivileges=yes
PrivateTmp=no
InaccessiblePaths=/root /sys /srv /media -/lost+found
ReadWritePaths=/opt/minecraft

ExecStart=/opt/minecraft/server/run.sh
ExecStop=/opt/minecraft/server/stop.sh
Restart=always
RestartSec=10
```

This isn't a systemd tutorial so I won't be explaining in great detail what everything here does, but it's mostly self-explanatory. The middle section of the ```[Service]``` stanza contains a variety of options to help enhance the security of our system in case the Minecraft server is compromised.

Now that our systemd unit file is written, we can (as root) run ```systemctl enable --now minecraft.service``` to start the server and enable it at boot. We can now manage our server with the typical ```systemctl``` commands such as ```stop```, ```start```, ```restart```, etc.


# Allow others to connect

The final step is to configure our system firewall to allow external connections to Minecraft. To do this we just need to open TCP port 25565.

```
# firewall-cmd --permanent --add-port 25565/tcp
# firewall-cmd --reload
```

### All done!

That's it! We now have a fully functional Minecraft server for cozy gaming with friends. From here, you may want to refer to the Minecraft wiki to configure things such as the server whitelist, server operators, MOTD, etc.

We can make backups of your server's worldfile by simply compressing an archive of the ```world``` directory into the ```/opt/minecraft/backup``` folder we created earlier. If you're interested in getting automated backups setup, consider using a tool like [Restic](https://restic.net/).
