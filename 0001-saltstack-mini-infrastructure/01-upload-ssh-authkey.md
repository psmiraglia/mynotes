# Upload SSH auth key on the minions

This is an example that shows how to use SaltStack in order to upload on all
the minions the SSH pubkey of a new administrator (Alice).

The write up is based on the infrastructure created by following the
[basic installation guide](./00-basic-installation.md).

## Preparing the `master`

Create a local directory to store the pubkeys to be uploaded.

    $ mkdir -p /opt/salt/ssh-keys

Then copy the Alice's pubkey inside it.

    $ cp alice.pub /opt/salt/ssh-keys

Restart the `master`.

    $ service salt-master restart

## Upload the Alice's pubkey

For this task, I'm using the SaltStack
[SSH module](https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.ssh.html).

    $ salt '*' ssh.set_auth_key_from_file root salt://ssh-keys/alice.pub
    minion-1:
        new
    minion-2:
        new

To check which keys are currently configured on the `minions`

    $ salt '*' ssh.auth_keys
    minion-1:
    ----------
    root:
        ----------
        AAAAB3NzaC [...] MYMnMzG52n:
            ----------
            comment:
                alice
            enc:
                ssh-rsa
            fingerprint:
                d5:f7:ef:91:85:e3:00:a9:5e:33:7b:87:80:43:c8:87
            options:
    minion-2:
    ----------
    root:
        ----------
        AAAAB3NzaC [...] MYMnMzG52n:
            ----------
            comment:
                alice
            enc:
                ssh-rsa
            fingerprint:
                d5:f7:ef:91:85:e3:00:a9:5e:33:7b:87:80:43:c8:87
            options:
