UDR
===

[![Build Status](https://travis-ci.org/LabAdvComp/UDR.svg?branch=master)](https://travis-ci.org/LabAdvComp/UDR)

UDR is a wrapper around rsync that enables rsync to use the UDP-based Data Transfer Protocol (UDT).

This is a copy of the original UDR code, and adds an option to limit the bandwidth at the UDT layer.

CONTENT
-------
./src:     UDR source code
./udt:	   UDT source code, documentation and license

TO MAKE
-------
    make -e os=XXX arch=YYY

XXX: [LINUX(default), BSD, OSX]   
YYY: [AMD64(default), POWERPC, IA64, IA32]  

### Dependencies:
OpenSSL (Debian: libssl and libcrypto, CentOS: openssl-devel)
Currently, UDR has mainly been tested on Linux so your mileage may vary on another OS. UDT has been well tested on all of the provided options.

USAGE
------
UDR must be on the client and server machines that data will be transferred between. UDR uses ssh to do authentication and automatically start the server-side UDR process. At least one UDP port needs to be open between the machines, by default UDR starts with port 9000 and looks for an open port up to 9100, changing this is an option. Encryption is off by default. When turned on encryption uses OpenSSL with aes-128 by default.

### Basic usage:
    udr [udr options] rsync [rsync options] src dest

### UDR options:

- `[-a start port] UDT port
- `[-b end port] UDT port
- `[-c path] Explicit path to remote UDR executable, if not in user path
- `[-d timeout] Data transfer timeout in seconds, default is 15s
- `[-i ip]` specify the interface by ip that the remote process will bind to
- `[-n aes-128 | aes-192 | aes-256 | bf | des-ede3]` turns on encryption, if crypto is not specified aes-128 is the default
- `[-o server port] Port to access a UDR server, default 9000
- `[-p path]` local path for the .udr_key file used for encryption, default is the current directory
- `[-P ssh-port] Remote port to connect to via SSH
- `[-r max-bw] Maximum bandwidth to utilize (Mbps)
- `[-v] Run UDR with verbosity
- `[--version]` print out the version

The rsync [rsync options] should take any of the standard rsync options, except the -e/--rsh flag which how UDR interfaces with rsync.

### A basic example command:
    udr rsync -av --stats --progress /home/user/tmp/ hostname.com:/home/user/tmp

### A command with udr options:
    udr -c /home/user/udr/src/udr -a 8000 -b 8010 \
       rsync -av --stats --progress /home/user/tmp/ hostname.com:/home/user/tmp

    udr -a 2620 -b 2620 -r 950 -c /home/oper/bin/udr \
       rsync -av --stats --progress oper@vlbi-control1.iram.es:/tmp/random.vdif /data/testpv/

    udr -a 2620 -b 2620 -r 950 -c /home/oper/bin/udr \
       rsync -av --stats --progress --bwlimit=1160 oper@vlbi-control1.iram.es:/tmp/random.vdif /data/testpv/


### Notes:
After the rsync data transfer is complete, the local udr thread is shutdown by a signal. Rsync thinks this is abnormal and prints out the error "rsync error: sibling process terminated abnormally", which can be ignored. However, the transfer should be complete, if other rsync errors appear these are true errors.

UDR SERVER
----------
The UDR server allows UDR transfers for users without accounts, similar to rsync server functionality. The UDR server is written in python, listens on a TCP port and mainly manages launching rsync processes with the "using rsync-daemon features via a remote-shell connection" ability of rsync (see rsync man page for details). The UDR server requies UDR version 0.9.2 or above.

### Basic server usage:
    python udrserver.py [-v] [-s] [-c configfile] start|stop|restart|foreground

### UDR server options:
- `[-c config file]` specify the location of the config file, default is /etc/udrd.conf
- `[-s]` silent mode, don't print message on start|stop|restart
- `[-v]` verbose mode, mainly for debugging purposes

### UDR server configuration:
The UDR server requires a configuration file, by default it looks for /etc/udrd.conf. The format of the file is a list of parameter of the format 'name = value'. An example config file is provided, the available parameters are:

- address: IP address to bind to, default is 0.0.0.0
- server port: TCP port for the server to listen on, default is 9000
- start port: first UDP port to begin UDR connections on, default is 9000
- end port: last UDP port to begin UDR connections on, default is 9100
- log file: log file for UDR, default is <current working dir>/udr.log
- log level: level of logging used, based on the python logging module, default is INFO
- pid file: pid file used for daemon, default is `/var/run/udrd.pid`
- udr: path to udr command, default is udr
- rsyncd conf: rsyncd.conf file to use for the rsync part of the configuration
- uid: user name or uid that the server should run as when started as root, default is nobody when run as root
- gid: group name or gid that the server should run as when started as root, default is nogroup when run as root
- specify ip: IP address for udr receiver to bind to, default is any connected interface

Most standard `rsyncd.conf` options should work like normal. Known exceptions are:

#### Max Connections
The max connections option does not work, but the number of connections can be limited by the range of start and end port in the udrd.conf file because one connection requires one port. For example, if you only want 50 connections, set the start port to 9000 and the end port to 9050. If the max connections option is set, the rsync processes will try to write to the lock file, which they often do not have permission to and will return the error "@ERROR: failed to open lock file". You can set the lock file location in `rsyncd.conf` or remove the max connections option.

#### UID/GID and chroot
It is not recommended to run UDR server as root. However, then the rsync use chroot option will not be available. If chroot is desired, the uid/gid options in udrd.conf must be explicitly set to root. UDR will then parse and use the global uid/gid settings in `rsyncd.conf` for spawned subprocesses. However, it does not currently support different uid/gid for each module.

#### WARNING: UDR server has only be tested in read only mode, it is not recommended to enable write access.

### Connecting to the UDR server
To connect to the UDR server, use double colons instead of the single colon, similar to connecting to a rsync daemon. Listing files is also the same as with rsync.

### Basic example command for downloading from a UDR server:
    udr rsync -av --stats --progress hostname.com::module/path/to/file /home/user/target

### List modules available:
    udr rsync hostname.com::

### List files on server:
    udr rsync hostname.com::module/path/to/file
