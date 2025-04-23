## nftblocklist
*nftblocklist* is a script and associated config files that will allow you to maintain a persistent-across-restart IPv4 blocklist using `nftables` on Ubunutu 22+. 

### DISCLAIMER and LICENSE
I am not a firewall engineer, nor am I a developer; this script and its associated files are a 'works for me' solution.  I post it in the hopes that it may be useful to others who, like me, went to the internet looking for recommendations on how to manage IP blocking on their nftables installation and to have those blocklists maintained across service and system restarts.
  
This software is licensed under the license contained in this repo at /LICENSE.  The contents have been assembled from various internet answers as well as some light and hopefully not regrettable original scripting by myself.

  
### Repo Contents
The repository contains the primary script (`nftblocklist`) as well as configuration files.  The configuration files are meant to permit modification of the `nftables` systemd entry to support the nftblocklist workflow.


##### nftblocklist
This bash script performs the actual operations.  It has a `-h` flag which will provide current capabilities and syntax.  It is meant to be installed in `/usr/sbin` or similar.  If used with the `-v` flag, it will output step and progress information.

This script has its operations configured via variables, which are labeled and identified within the script.  You are encouraged to test and configure the script to meet your `nftables` setup requirements (filter names, set names, etc.) which you can do using flags to specify them, and then modify the defaults in the script to permit bare invocation of the script to work.  By default, the script will only output messages of `err` or 'lower' (more important) loglevel; adding '-v' will increase verbosity.

I encourage you to look at the script's contents for more information on what it's doing and how.


##### nftables.conf
This file is an example config file for `nftables` which (on Ubuntu 22.04, at least) lives at `/etc/nftables.conf`.  This configuration will set up nftables to deliver the basic functionality described below under [Default Functionality].  Using this file should guarantee that the script's default values function as intended.


##### nftstart / nftstop
These are simple scripts that should be placed in `/usr/sbin/` which should be called by systemd when nftables is started, stopped or reloaded.  These scripts are referenced in nftables.service, and are invoked using `ExecStart=`, `ExecStop=` and `ExecReload=`.

##### nftblocklist.cron.daily
A sample crontab file that can be placed in `/etc/cron.daily` to implement the scheduled functionality described in [Default Functionality].