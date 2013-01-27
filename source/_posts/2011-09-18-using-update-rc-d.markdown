---
layout: post
title: "Using update-rc.d"
date: "2011-09-18 21:00:00"
comments: true
categories: daemon update-rc unix linux
---

`update-rc.d` is tool for adding daemons to /etc/rc[1-6]d files.

If you are using OS with System-V style init scripts you can find you `/etc` the next directories:

* rc0.d/
* rc1.d/
* rc2.d/
* rc3.d/
* rc4.d/
* rc5.d/
* rc6.d/
* rcS.d/

So, what are they for?


<!--more-->

Every directory contains symbol links to daemons located in `/etc/init.d/`. And every directory represents run level. Usually your system uses third level which means multi user mode with networking. When the system is initializing it starts daemons which have appropriate symbol links in rc3.d.

Here is a short description of every run level:


* __0 (Halt)__ 	Shuts down the system.
* __1 (Single-User Mode)__ 	Mode for administrative tasks.[2][3]
* __2 (Multi-User Mode)__ 	Does not configure network interfaces and does not export networks services.[4]
* __3 (Multi-User Mode with Networking)__ 	Starts the system normally.[5]
* __4 (Not used/User-definable)__ 	For special purposes.
* __5 (Start the system normally with appropriate display manager.)__ 	As runlevel 3 + display manager.
* __6 (Reboot)__ 	Reboots the system.

Usualy you don't need to switch levels, but if you do you should use `init` command. Take care of what you doing.

OK, take a look at a head of some daemon scripts. For example `/etc/init.d/postgresql`:

```bash /etc/init.d/postgresql

#!/bin/sh
set -e

### BEGIN INIT INFO
# Provides:             postgresql
# Required-Start:       $local_fs $remote_fs $network $time
# Required-Stop:        $local_fs $remote_fs $network $time
# Should-Start:         $syslog
# Should-Stop:          $syslog
# Default-Start:        2 3 4 5
# Default-Stop:         0 1 6
# Short-Description:    PostgreSQL RDBMS server
### END INIT INFO

```

It has a self descriptive comment in an appropriate format. `Default-Start` section contains numbers of levels which runs postgres daemon. `Default-Stop` section contains number of levels which should not run PostgreSQL, so when you switch to them PostgreSQL daemon will be stopped.

So, let's assume you have created you own daemon which knows how to process `start` and `stop` arguments and want it to run on appropriate levels. You need to create symbol links like these:

    /etc/rc1.d/K03postgresql -> ../init.d/postgresql
    /etc/rc4.d/S02postgresql -> ../init.d/postgresql
    /etc/rc0.d/K03postgresql -> ../init.d/postgresql
    /etc/rc2.d/S02postgresql -> ../init.d/postgresql
    /etc/rc3.d/S02postgresql -> ../init.d/postgresql
    /etc/rc6.d/K03postgresql -> ../init.d/postgresql
    /etc/rc5.d/S02postgresql -> ../init.d/postgresql

By the way prefix `K` means kill(stop) process and `S` means start.
It's a little routine work, right? But in case if you have descriptive comment I metioned above you just can use `update-rc.d` tool. It creates all necessary symbol links based on the comment. So just run:

    update-rc.d postgres defaults

Change `postgres` with name of your daemon.

## Why?

Unfortunatly I often forget how this tool is called.

