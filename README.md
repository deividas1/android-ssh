# How to control Android phone's ssh tunnel with SMS messages using Termux and Automate

## Introduction

In this tutorial replace all addresses `101.102.103.104` with your `server address` and `192.168.0.10` with `Android's private address`.

## App installation

Our goal is to be able to start and stop ssh tunnel to Linux server from Android phone. If phone uses mobile data there is no way to ssh to it other than via ssh tunnel. To achieve this goal we will use [Automate](https://play.google.com/store/apps/details?id=com.llamalab.automate) app which can listen for SMS messages and will start or stop ssh tunnel. We will also need to install command line tool `Termux`. You can download it from [F-Droid](https://f-droid.org/F-Droid.apk) by choosing `Terminal emulator with packages` or from [here](https://f-droid.org/en/packages/com.termux/). Termux from `Google Play` is not suitable because Google Play's version has different limited functionality. During installation `Google Play Protect` will warn that `Termux` is unsafe app. Just ignore that. Click on `More details` and `Install anyway`. Next we will need [Termux:Tasker](https://f-droid.org/en/packages/com.termux.tasker/) which is Termux plugin that can be configured by `Automate`.

## Package installation

Run command which will install required packages

    apt update
    apt install -y openssh net-tools
    sshd    
    mkdir ~/.termux/tasker
    
    termux-setup-storage

## Script file

Run script below to create `smstask.sh` file in `/data/data/com.termux/files/home/.termux/tasker`. It will be called by `Termux:Tasker` plugin.

	cat > /data/data/com.termux/files/home/.termux/tasker/smstask.sh <<'EOF'
	#!/data/data/com.termux/files/usr/bin/bash

	server="root@101.102.103.104"
	server_key="/data/data/com.termux/files/home/.ssh/server"
	remote_port="25025"
	authorized_phones="37060020020 37060020021"
	log_file="/data/data/com.termux/files/home/.termux/tasker/smstask.log"

	function log_action() {
	  time1=$(date '+%Y-%m-%d %H:%M:%S')
	  echo "$time1"" \"""$cmd1""\" from ""$phone" >> "$log_file"
	}

	while getopts ":n:c:" opt; do
	 case $opt in
	  n)
	   phone="$OPTARG"
	   ;;
	  c)
	   cmd1=$(echo "$OPTARG" | awk '{print tolower($0)}')
	   ;;
	 esac
	done

	if [[ ! $authorized_phones =~ "$phone" ]]; then
	  time1=$(date '+%Y-%m-%d %H:%M:%S')
	  echo "$time1"" Phone ""$phone"" is unauthorized to run ""\"""$cmd1""\"" >> "$log_file"
	  exit
	fi

	if [[ $cmd1 =~ "ssh on" ]]; then
	  termux-wake-lock
	  sshd
          ssh "$server" -i "$server_key" 'ids=$(lsof -t -i:'"$remote_port"'); if [ "$ids" ]; then kill $ids; fi'
	  ssh -f -N -o "ServerAliveInterval 60" -R "$remote_port":localhost:8022 "$server" -i "$server_key"
	  log_action
	fi

	if [[ $cmd1 =~ "ssh off" ]]; then
	  ssh "$server" -i "$server_key" 'ids=$(lsof -t -i:'"$remote_port"'); if [ "$ids" ]; then kill $ids; fi'
	  log_action
	fi
	EOF

	chmod +x /data/data/com.termux/files/home/.termux/tasker/smstask.sh

Set correct script variables:

    nano ~/.termux/tasker/smstask.sh
    
    server="root@101.102.103.104"
    server_key="/data/data/com.termux/files/home/.ssh/server"
    remote_port="25025"
    authorized_phones="37060020020 37060020021"
    log_file="/data/data/com.termux/files/home/.termux/tasker/smstask.log"

Next step is to add server's fingerprint to `known_hosts`

     ssh-keyscan -t ed25519 101.102.103.104 >> ~/.ssh/known_hosts

## Creating ssh key on Android
At first you can ssh to your phone with password. For that set the password using command `passwd` and run ssh server with `sshd` command. Connect to phone with command:

    ssh -p 8022 root@192.168.0.10

Now let's copy and paste script to terminal:

    cd ~/.ssh
    ssh-keygen -t ecdsa-sha2-nistp256 -N "" -C "android" -f android
    cat android.pub >> authorized_keys
Set permissions

    chmod 700 ~/.ssh
    chmod 600 android
    chmod 644 android.pub
    chmod 600 authorized_keys

## Copy key to your PC with USB cable

To copy `android` file to PC, connect phone to PC with USB cable.
Command `termux-setup-storage` allows to access phone files from Termux. Once it is run, it will ask permission and create `~/storage directory`. Next copy key files with commands

    mkdir ~/storage/shared/termux_files
    cp ~/.ssh/android ~/storage/shared/termux_files/

## Copy key with scp

Another way to copy file is to use scp command.
First find out Android's ip with `ifconfig` command. 

    scp -P 8022 root@192.168.0.10:~/.ssh/android android
After key file is copied turn off passwords in ssh configuration

    sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' $PREFIX/etc/ssh/sshd_config
And restart ssh server from your phone's terminal

    pkill sshd; sshd

## Server's ssh configuration

Connect to server via local console and run these commands

    sed -i "s/.*PermitRootLogin.*/PermitRootLogin yes/" /etc/ssh/sshd_config
    sed -i "s/.*GatewayPorts.*/GatewayPorts yes/" /etc/ssh/sshd_config
    service ssh restart
    
    cd ~/.ssh
    ssh-keygen -t ecdsa-sha2-nistp256 -N "" -C "server" -f server
    cat server.pub >> authorized_keys

    chmod 600 server
    chmod 644 server.pub
    chmod 600 authorized_keys
    
    cat server

Copy `server` content from local terminal and run command on Android terminal

    echo "-----BEGIN OPENSSH PRIVATE KEY-----
    ...
    ...
    -----END OPENSSH PRIVATE KEY-----" > ~/.ssh/server
    
    chmod 600 ~/.ssh/server
    
## Automate configuration

Create flow as shown in the image

[<img src="images/image1.jpg" width="300" />](images/image1.jpg)

In SMS Received block set Output variables `phone1` and `text1`

[<img src="images/image2.jpg" width="300" />](images/image2.jpg)

To find **Termux:Tasker** search by word `action` and choose `Plug-in action`

[<img src="images/image3.jpg" width="300" />](images/image3.jpg)

Next configure Tasker block as follows

[<img src="images/image4.jpg" width="300" />](images/image4.jpg)

and launch the flow

[<img src="images/image5.jpg" width="300" />](images/image5.jpg)

Now you can control ssh tunnel by texting SMS commands `ssh on` and `ssh off` to your phone. When ssh tunnel is up and running you can ssh to your Android from anywhere with command:

    ssh -p 25025 root@101.102.103.104 -i ~/.ssh/android 

As long as Automate flow will be running, SMS functionality will work.
