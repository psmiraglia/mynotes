# SaltStack mini infrastructure

This guide explains how to setup a simple SaltStack infrastructure composed of
one `master` and two `minions`. The staring point is a minimal installation of
Debian 8.4 (amd64).

## Common steps

Apply the following steps to all the servers composing your infrastructure.

### Adding APT repositories

    $ cd /etc/apt/source.list.d
    $ echo "deb http://debian.saltstack.com/debian jessie-saltstack main" > saltstack.list
    $ wget -q -O- "http://debian.saltstack.com/debian-salt-team-joehealy.gpg.key" | apt-key add -
    $ apt-get update

## Salt `master`

Apply the following steps only on the `master`.

### Setup the firewall

On the `minions`, no incoming connections should be allowed. On the contrary,
the `master` needs TCP ports 4505 and 4506 to be opened. See
[Opening the Firewall up for Salt](https://docs.saltstack.com/en/latest/topics/tutorials/firewall.html)
for more details.

Create a new chain named `SALTSTACK` and append (or insert) it to the system
`INPUT` chain

    $ iptables -N SALTSTACK
    $ iptables -A INPUT -j SALTSTACK

Populate the `SALTSTACK` chain with the following rules (we assume `minions`
operating on `192.168.100.0/24` and `192.168.101.0/24` networks).

    $ iptables -A SALTSTACK -i lo -p tcp -m multiport --dports 4505,4506 -j ACCEPT
    $ iptables -A SALTSTACK -s 192.168.122.0/24 -p tcp -m state --state new -m multiport --dports 4505,4506 -j ACCEPT
    $ iptables -A SALTSTACK -s 192.168.123.0/24 -p tcp -m state --state new -m multiport --dports 4505,4506 -j ACCEPT

### Install the `master`

    $ apt-get update
    $ apt-get install salt-master

### Configure the `master`

SaltStack supports a modular configuration. Settings may be specified in
separate `.conf` files using YAML syntax.
Conf files are located under `/etc/salt/master.d` directory. Of course, it may
change according to the distribution.

For the purpose of this guide, we use two files named `primary.conf` and
`file_server.conf`. The former includes main settings

    $ cat /etc/salt/master.d/primary.conf
    interface: 192.168.122.146
    ipv6: False

while the latter, configures the local file server.

    $ cat /etc/salt/master.d/file_server.conf
    file_roots:
      base:
        - /srv/salt

Further details about the `master` configuration on the
[official guide](https://docs.saltstack.com/en/latest/ref/configuration/master.html).

Before restarting the `master`, create (if not exists) all the paths declared
under `file_roots` directive. In this case

    $ mkdir /srv/salt

Restart the `master`

    $ service salt-master restart

## Salt `minion`

Apply the following steps on all the `minions`.

### Install the `minion`

    $ apt-get update
    $ apt-get install salt-minion

### Configure the `minion`

As for the `master`, also `minions` configuration can be modular.
Configuration files are located under `/etc/salt/minon.d`.

Firstly, we need to specify who is the `master`. You may use `IP` or
`hostname`.

    $ cd /etc/salt/minion.d
    $ echo "master: salt-master" > primary.conf

Then, for security reasons, we need to specify the `master`'s fingerprint.
To get this value, run on the `master` the following command

    $ salt-key -F master
    Local Keys
    master.pem:  8c:f6:e6:4f:4a:45:24:9e:b3:3d:8e:e0:76:9e:ca:70
    master.pub:  6c:3c:19:ec:09:1d:36:9f:d4:d3:be:1f:e3:57:71:dc
    ...

Grab the `master.pub` value and put in in the `minion` configuration

        $ cd /etc/salt/minion.d
        $ echo "master_finger: '6c:3c:19:ec:09:1d:36:9f:d4:d3:be:1f:e3:57:71:dc'" > security.conf

Further details about the `minion` configuration on the
[official guide](https://docs.saltstack.com/en/latest/ref/configuration/minion.html).

## Back to the salt `master`

We're quite ready. Before proceeding, we need to register the `minions` on the
`master`. To show the available `minions` keys, run the command

    $ salt-key -L
    Accepted Keys:
    Denied Keys:
    Unaccepted Keys:
    salt-minion-1
    salt-minion-2
    Rejected Keys:

As you can see, there are two unaccepted keys. You can accept all in one shot
using

    $ salt-key -A

or singularly by using

    $ salt-key -a <key-id>

Once accepted, you should have something like that

    $ salt-key -L
    Accepted Keys:
    salt-minion-1
    salt-minion-2
    Denied Keys:
    Unaccepted Keys:
    Rejected Keys:

Let's start with the first test

    $ salt '*' test.ping
    salt-minion-2:
        True
    salt-minion-1:
        True

It seems to work...
