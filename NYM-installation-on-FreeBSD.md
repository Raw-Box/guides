# NYM installation on FreeBSD


# Prerequisites

1.0) If running on VMware platform _*optional_
```
pkg install open-vm-tools-nox11
```

1.1) Update the local package repository database & upgrade installed packages to their latest versions based on the updated package database
```
pkg update && pkg upgrade
```

1.2) To enable the Linux ABI at boot time, execute the following command:
```
sysrc linux_enable="YES"
```

1.3) Once enabled, it can be started without rebooting executing the following command:
```
service linux start
```

1.4) Install packages:
```
pkg install pkgconf nano curl jq git openssl libressl gmake gcc
```

1.5) Install Rust:
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

When you see next message, press "1":

![Screenshot 2024-02-05 215537](https://github.com/crptcpchk/RawBox/assets/94843482/41d14f99-ab4b-47e5-a534-0ff82a4fe25d)

1.6) Add the Rust binaries to your system PATH:
```
. "$HOME/.cargo/env"
```

1.7) Verify that Rust is installed by checking the version:
```
rustc --version
```

1.8) If needed, update rustup:
```
rustup update
```

# Download and build NYM binaries

2.1) Clone git:
```
git clone https://github.com/nymtech/nym.git
```

2.2) Move to `nym` folder:
```
cd nym
```

2.2.1) In case you haven't update git for a while:
```
git pull
```

2.3) Switch to `develop` branch

`develop` branch will most likely be incompatible with deployed public networks, but for now `master` branch uses `boringtun` which doesn't build on FreeBSD and ends up with _error E0599_.
```
git checkout develop
```

2.4) Build your binaries:
```
cargo build --release
```

# Mix node setup

3.1) Move to `$HOME/nym/target/release/` folder. You will run commands from this folder.
```
cd target/release
```
3.2) Initialize your mix node:
```
./nym-mixnode init --id <YOUR_ID> --host $(curl -4 https://ifconfig.me) 
```
Change `<YOUR_ID>` with preferred ID name;

`--host $(curl -4 https://ifconfig.me)` command will automatically return your IP address.

If everything went ok, you should see similar message:

![Screenshot 2024-02-08 095547](https://github.com/crptcpchk/RawBox/assets/94843482/99ba90e2-0f83-4630-a5c2-f681417924ef)

3.3) Node Description _*optional_
You can add description to your node, so people can easily identify it in explorers.
```
./nym-mixnode describe --id <YOUR_ID>
```
Where _<YOUR_ID>_ is an ID you specified in step 3.2.

Example. The description consist of the following information:
```
name = "RawBox Privacy Router"
description = "Privacy issues aren't solely addressed through the mixnet. First, there needs to be a private way to connect to it."
link = "https://github.com/Raw-Box"
location = "Ukraine"
```

# Running mix node with `rc` service file

4.1) Move to `rc.d` folder:
```
cd /usr/local/etc/rc.d/
```

4.2) Create `nym-mixnode` file:
```
nano nym-mixnode
```

4.3) Copy/paste this code:
```
#!/bin/sh

# PROVIDE: nym_mixnode
# REQUIRE: LOGIN
# KEYWORD: shutdown

. /etc/rc.subr

name="nym_mixnode"
rcvar="nym_mixnode_enable"

load_rc_config $name

: ${nym_mixnode_enable:="NO"}
: ${nym_mixnode_user:="<USER>"}
: ${nym_mixnode_path:="/home/<USER>/<PATH>"}
: ${nym_mixnode_id:="<YOUR_ID>"}

command="/usr/sbin/daemon"
command_args="-u $nym_mixnode_user -p /var/run/$name.pid $nym_mixnode_path/nym-mixnode run --id $nym_mixnode_id"

start_cmd="${name}_start"
stop_cmd="${name}_stop"

nym_mixnode_start() {
    if [ ! -f /var/run/$name.pid ]; then
        install -o $nym_mixnode_user /dev/null /var/run/$name.pid
        chown $nym_mixnode_user /var/run/$name.pid
    fi

    $command $command_args
}

nym_mixnode_stop() {
    kill `cat /var/run/$name.pid` 2>/dev/null
}

run_rc_command "$1"
```
Where:

`<USER>` must be change for your username or root, if your building under root;

`/home/<USER>/<PATH>` - must be changed for your working directory. In case of root it's a `/root/nym/target/release`

`<YOUR_ID>` is an ID you specified in step 3.2.

4.3.1) Save the file:
```
Crtl+X
Y
Enter
```

4.4) Make the script executable:
```
chmod +x /usr/local/etc/rc.d/nym-mixnode
```
4.5) Move to `/etc/` folder:
```
cd /etc/
```

4.6) Open `rc.conf` file:
```
nano rc.conf
```

4.7) Add next line to the file:
```
nym_mixnode_enable="YES"
```

4.7.1) Save the file:
```
Crtl+X
Y
Enter
```

4.8) Start and stop nym-mixnode:

Start:
```
service nym-mixnode start
```

Stop:
```
service nym-mixnode stop
```

# Network Requester setup

5.1) Move to `$HOME/nym/target/release/` folder. You will run commands from this folder.
```
cd target/release
```
5.2) Initialize your network requester:
```
./nym-network-requester init --id <YOUR_ID>
```

Change `<YOUR_ID>` with preferred ID name;

If everything went ok, you should see similar message:

![Screenshot 2024-02-09 114158](https://github.com/crptcpchk/RawBox/assets/94843482/9954bbfd-74a1-4135-ba61-f675148583df)

# Using your Network Requester

The next thing to do is use your requester, share its address with friends (or whoever you want to help privacy-enhance their app traffic). 

Is this safe to do? If it was an open proxy, this would be unsafe, because any Nym user could make network requests to any system on the internet.

To make things a bit less stressful for administrators, the Network Requester drops all incoming requests by default. In order for it to make requests, you need to add specific domains to the `allowed.list` file at `$HOME/.nym/service-providers/network-requester/allowed.list` or if Network Requester is ran as a part of **Exit Gateway**, the `allowed.list` will be stored in `~/.nym/gateways/<ID>/data/network-requester-data/allowed.list`.

# Running Network Requester with `rc` service file

6.1) Move to `rc.d` folder:
```
cd /usr/local/etc/rc.d/
```
6.2) Create nym-network-requester file:
```
nano nym-network-requester
```
6.3) Copy/paste this code:
```
#!/bin/sh

# PROVIDE: nym_network_requester
# REQUIRE: LOGIN
# KEYWORD: shutdown

. /etc/rc.subr

name="nym_network_requester"
rcvar="nym_network_requester_enable"

load_rc_config $name

: ${nym_network_requester_enable:="NO"}
: ${nym_network_requester_user:="<USER>"}
: ${nym_network_requester_path:="/home/<USER>/<PATH>"}
: ${nym_network_requester_id:="<YOUR_ID>"}

command="/usr/sbin/daemon"
command_args="-u $nym_network_requester_user -p /var/run/$name.pid $nym_network_requester_path/nym-network-requester run --id $nym_network_requester_id"

start_cmd="${name}_start"
stop_cmd="${name}_stop"

nym_network_requester_start() {
    if [ ! -f /var/run/$name.pid ]; then
        install -o $nym_network_requester_user /dev/null /var/run/$name.pid
        chown $nym_network_requester_user /var/run/$name.pid
    fi

    $command $command_args
}

nym_network_requester_stop() {
    kill `cat /var/run/$name.pid` 2>/dev/null
}

run_rc_command "$1"
```

Where:

`<USER>` must be change for your username or root, if your building under root;

`/home/<USER>/<PATH>` - must be changed for your working directory. In case of root it's a /root/nym/target/release

`<YOUR_ID>` is an ID you specified in step 5.2.

6.3.1) Save the file:
```
Crtl+X
Y
Enter
```
6.4) Make the script executable:
```
chmod +x /usr/local/etc/rc.d/nym-network-requester
```
6.5) Move to `/etc/` folder:
```
cd /etc/
```
6.6) Open `rc.conf` file:
```
nano rc.conf
```
6.7) Add next line to the file:
```
nym_network_requester_enable="YES"
```
6.7.1) Save the file:
```
Crtl+X
Y
Enter
```
6.8) Start and stop nym-network-requester:

Start:
```
service nym-network-requester start
```

Stop:
```
service nym-network-requester stop
```

# Exit gateway setup

7.1) Move to `$HOME/nym/target/release/` folder. You will run commands from this folder.
```
cd target/release
```
7.2) Initialise your gateway:
```
./nym-gateway init --id <ID> --listening-address 0.0.0.0 --public-ips "$(curl -4 https://ifconfig.me)" --with-network-requester --with-exit-policy true
```
Where:

Change `<ID>` with preferred ID name;

`--listening-address` - is the IP address which is used for receiving sphinx packets and listening to client data;

`--public-ips` - it’s a comma separated list of IP’s that are announced to the nym-api, it is usually the address which is used for bonding;

`--with-network-requester` - means you are running a network-requester in the same process as the gateway process;

`--with-exit-policy true` - this flag sets up Exit Gateway functionality with NYM [exit policy](https://nymtech.net/.wellknown/network-requester/exit-policy.txt)

If everything went ok, you should see similar message:

![Screenshot 2024-02-08 105146](https://github.com/crptcpchk/RawBox/assets/94843482/935639fb-10fd-4ca5-9450-1afd09a13667)

# Running Exit Gateway with `rc` service file

8.1) Move to `rc.d` folder:
```
cd /usr/local/etc/rc.d/
```
8.2) Create nym-gateway file:
```
nano nym-gateway
```
6.3) Copy/paste this code:
```
#!/bin/sh

# PROVIDE: nym_gateway
# REQUIRE: LOGIN
# KEYWORD: shutdown

. /etc/rc.subr

name="nym_gateway"
rcvar="nym_gateway_enable"

load_rc_config $name

: ${nym_gateway_enable:="NO"}
: ${nym_gateway_user:="<USER>"}
: ${nym_gateway_path:="/home/<USER>/<PATH>"}
: ${nym_gateway_id:="<YOUR_ID>"}

command="/usr/sbin/daemon"
command_args="-u $nym_gateway_user -p /var/run/$name.pid $nym_gateway_path/nym-gateway run --id $nym_gateway_id"

start_cmd="${name}_start"
stop_cmd="${name}_stop"

nym_gateway_start() {
    if [ ! -f /var/run/$name.pid ]; then
        install -o $nym_gateway_user /dev/null /var/run/$name.pid
        chown $nym_gateway_user /var/run/$name.pid
    fi

    $command $command_args
}

nym_gateway_stop() {
    kill `cat /var/run/$name.pid` 2>/dev/null
}

run_rc_command "$1"
```

Where:

`<USER>` must be change for your username or root, if your building under root;

`/home/<USER>/<PATH>` - must be changed for your working directory. In case of root it's a /root/nym/target/release

`<YOUR_ID>` is an ID you specified in step 5.2.

6.3.1) Save the file:
```
Crtl+X
Y
Enter
```
6.4) Make the script executable:
```
chmod +x /usr/local/etc/rc.d/nym-gateway
```
6.5) Move to `/etc/` folder:
```
cd /etc/
```
6.6) Open `rc.conf` file:
```
nano rc.conf
```
6.7) Add next line to the file:
```
nym_gateway_enable="YES"
```
6.7.1) Save the file:
```
Crtl+X
Y
Enter
```
8.8) Start and stop nym-gateway:

Start:
```
service nym-gateway start
```

Stop:
```
service nym-gateway stop
```
