# nmtrust

This project provides a simple framework for determining the trusted state of the
current network connections, and taking action based on the result. It is
intended to be used to activate certain services on trusted networks, and
disable them when when there is a connection to an untrusted network or when
there is no established network connection.

## Requirements

* [NetworkManager](https://wiki.gnome.org/Projects/NetworkManager)
* [sudo](https://www.sudo.ws/)

## Defining Trust

NetworkManager assigns a UUID to each network profile. These can be seen by
running `nmcli conn`. The UUIDs of the trusted networks should be placed in a
file. By default `nmtrust` will look for this file at
`/etc/nmtrust/trusted_networks`, however an alternative location may be
provided using the `-t` option.

If all of the current network connections are trusted, the trusted network file
can be initiated with these values.

    # nmcli --terse -f uuid conn show --active > /etc/nmtrust/trusted_networks

`nmtrust` will ask NetworkManager for a list of all active connections. It will
then compare the UUIDs of the active connections against the trusted network
file.

* If **all** the current connections are matched in the trusted network file,
  `nmtrust` will report that all connections are trusted.
* If **none** of the current connections exist in the trusted network file,
  `nmtrust` will report that all connections are untrusted.
* If **some** of the current connections exist in the trusted network file, but
  some do not, `nmtrust` will report that one or more connections are
  untrusted.
* If there are no active network connections, `nmtrust` will report this.

### Exclude Networks

Network connections can be excluded from nmtrust as well.

For example, if you have Docker installed, the Docker bridge network connection(s)
needs to be excluded from the active network connections list. Otherwise, if you
disconnect all other connections, `nmtrust` still thinks there are active connections
despite that you are offline.

The name of the network(s) that need to be excluded should be placed in
`/etc/nmtrust/excluded_networks`, however an alternative location may be
provided using the `-e` option.

You can place the exact names in the file or you can use wildcards to exclude multiple
networks. For example, `virbr0`, `virbr1`, etc. pp. or just `virbr?`. You can also
specify a range: `virbr[0,1]`.

### Usage

A unique exit code is returned for each of the four possible states.

Exit Code | State
--------- | -----
0         | All connections are trusted
2         | All connections are untrusted
3         | One or more connections are untrusted
4         | There are no active connections

This allows the user to easily script `nmtrust` to only execute certain actions
on certain types of networks. For example, you may have a network backup script
`netbackup.sh` that is executed every hour by cron. However, you only want the
script to run when you are connected solely to a network or networks that you
trust. This is easy to accomplish by creating a wrapper around `netbackup.sh`
for cron to call.

```
#!/bin/sh

# Execute nmtrust
nmtrust

# Execute backups if the current connection(s) are trusted.
if [ $? -eq 0 ]; then
    netbackup.sh
fi
```

## systemd integration

While `nmtrust` is a flexible script that can run anywhere NetworkManager is
present, `ttoggle` is provided for use on systems with
[systemd](https://wiki.freedesktop.org/www/Software/systemd/).

The idea here is that the user has a number of systemd units that they only
want to start when connected to a trusted network. The name of the trusted
units should be placed in a file, one per line. By default `ttoggle` will look
for this file at `/etc/nmtrust/trusted_units`, however an alternative location
may be provided using the `-f` option.

When `ttoggle` is executed, it calls `nmtrust` to determine the state of the
network connections. If `nmtrust` reports that all the current connections are
trusted, `ttoggle` will start all the units listed in the trusted unit file. If
`nmtrust` reports that there is a connection to an untrusted network or that
the system is offline, `ttoggle` will stop all the units listed in the trusted
unit file.

### Usage

The user may have a
[timer](http://www.freedesktop.org/software/systemd/man/systemd.timer.html) to
periodically send and receive mail, and a service that provides an IRC instant
messaging gateway. These may both potentially leak personal information over
the network, so they should not be started on untrusted connections.

    # echo 'mailsync.timer\nircgateway.service' > /etc/nmtrust/trusted_units

Now when `ttoggle` is called it will start or stop these trusted units as
appropriate.


#### Status

The `-s` option may be used to see an abbreviated status of all the trusted
units.

    $ ttoggle -s


#### Stop Everything

The `-x` option may be used to stop all of the trusted units, regardless of the
network trust.

    $ ttoggle -x


#### Start Everything

The `-t` option may be used to start all of the trusted units, regardless of
the network trust. This may be useful for temporarily trusting a network
connection.

    $ ttoggle -t


### Allow Offline

There may be some units that should be run on trusted networks *and* when there
is no network connection, but not when connected to an untrusted network. For
example, the [git-annex assistant](https://git-annex.branchable.com/assistant/)
provides useful functionality both online and offline, but may leak personal
information (such as the location of networked remotes) on untrusted networks.
These units can be allowed to run offline by adding `,allow_offline` to the
unit entry in the trusted unit file.

    # echo 'git-annex@user.service,allow_offline' >> /etc/nmtrust/trusted_units'

When `ttoggle` is called it will now perform the following:

* Start all units when connected to trusted networks.
* Stop all units when connected to untrusted networks.
* Stop all units when connected to no network, and then start units that are
  marked `allow_offline`.


### User Units

User units may be specified by adding `,user:username` to the unit entry in the
trusted unit file. For example, if the user `pigmonkey` has a unit
`ssh-tunnel.service` that should only be started on trusted networks:

    # echo 'ssh-tunnel.service,user:pigmonkey' >> /etc/nmtrust/trusted_units

When starting, stopping, or checking the status of these units `ttoggle` will
check if the calling user is the same as the user specified for the unit. If
the users match, the current user will be used to take the appropriate action.
If the users do not match (for instance, when `ttoggle` is called by root),
`sudo` will be used to take action as the specified user.

### Automation

A NetworkManager dispatcher is provided to automate the toggling of trusted
units. Once installed, the dispatcher will cause NetworkManager to call
`ttoggle` whenever a network connection is activated or deactived.

    # cp dispatcher/10trust /etc/NetworkManager/dispatcher.d
    # chmod 755 /etc/NetworkManager/dispatcher.d/10trust
