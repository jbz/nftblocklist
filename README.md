## nftblocklist
*nftblocklist* is a script and associated config files that will allow you to maintain a persistent-across-restart IPv4 blocklist using `nftables` on Ubunutu 22+. 

### DISCLAIMER and LICENSE
I am not a firewall engineer, nor am I a developer; this script and its associated files are a 'works for me' solution.  I post it in the hopes that it may be useful to others who, like me, went to the internet looking for recommendations on how to manage IP blocking on their nftables installation and to have those blocklists maintained across service and system restarts.
  
This software is licensed under the license contained in this repo at /LICENSE.  The contents have been assembled from various internet answers as well as some light and hopefully not regrettable original scripting by myself.


### Why nftblocklist?
After all, [netfilter-persistent](https://manpages.ubuntu.com/manpages/jammy/man8/netfilter-persistent.8.html) exists!  To be clear, this is not meant to replace that functionality, and in fact it conflicts slightly (or at least is slightly redundant) to that workflow.  `netfilter-persistent` is intended to preserve full firewall state in between reloads and system restarts.  This includes *all* functionality and rules - filters, chains, etc.  If you are running a very heavily dynamic firewall, which mutates its rules according to local conditions, or which multple humans are maintaining, this may be a solution for you.  `nftblocklist` is meant for a solo hobby server owner whose primary concern is 'the internet is always trying to break into my stuff, and I hate seeing a billion nmap or SSH bruteforce attempts in my log without doing *something*, and I don't modify my server config all that often.  I just want to be able to leave it there and have it do minimal responsive blocking using nftables.'  For a description of what `nftblocklist` offers out of the box, see [Default Functionality](#default-functionality) below.

  
### Repo Contents
The repository contains the primary script (`nftblocklist`) as well as configuration files.  The configuration files are meant to permit modification of the `nftables` systemd entry to support the nftblocklist workflow.


##### nftblocklist
This bash script performs the actual operations.  It has a `-h` flag which will provide current capabilities and syntax.  It is meant to be installed in `/usr/sbin` or similar.  If used with the `-v` flag, it will output step and progress information.

This script has its operations configured via variables, which are labeled and identified within the script.  You are encouraged to test and configure the script to meet your `nftables` setup requirements (filter names, set names, etc.) which you can do using flags to specify them, and then modify the defaults in the script to permit bare invocation of the script to work.  By default, the script will only output messages of `err` or 'lower' (more important) loglevel; adding '-v' will increase verbosity.

I encourage you to look at the script's contents for more information on what it's doing and how.


##### nftables.conf
This file is an example config file for `nftables` which (on Ubuntu 22.04, at least) lives at `/etc/nftables.conf`.  This configuration will set up nftables to deliver the basic functionality described below under [Default Functionality](default-functionality).  Using this file should guarantee that the script's default values function as intended.


##### nftstart / nftstop
These are simple scripts that should be placed in `/usr/sbin/` which should be called by systemd when nftables is started, stopped or reloaded.  These scripts are referenced in nftables.service, and are invoked using `ExecStart=`, `ExecStop=` and `ExecReload=`.

##### nftblocklist.cron.daily
A sample crontab file that can be placed in `/etc/cron.daily` to implement the scheduled functionality described in [Default Functionality].


### Default Functionality
The sample files configure a basic `nftables` firewall with default-deny policies and some specific permission stanzas.  The two capabilities that this config and this script provide involve automatic blocklisting both dynamic and persistent.  Dynamic blocklisting has an automatic timeout of 49 hours; persistent blocklisting is what this script handles.  Essentially, there are two conditions that trigger blocklisting:

1. Attempting to make an SSH connection more than 3 times within a minute from an IP other than the whitelisted IPs
2. Attempting to make more than 10 connections to a blocked port from an IP other than the whitelisted IPs

When these occur, the source ipv4 address is added to the set `blocklist_dyn` with a 49-hour timeout.  If no further action is taken, they will expire gracefully.

This is where the `nftblocklist` script comes in.

If this script is run with the -d (dump) option by an appropriately permissioned user (generally `root` or whichever user is managing the firewall) then the script will take the following actions:

1. Dump the current live nft ruleset as JSON, through `json_pp`, to a `jq` command which will isolate the element list of the `blocklist_dyn` set.  It pipes that through grep to come up with a list of single ipv4 addresses, one per line, and writes that to a tempfile.
2. It then appends the contents of the current persistent blocklist file (`/etc/nftables.blocklist` by default) to that tempfile.
3. It runs the tempfile as well as a second file - by default, `/etc/nftables.blockcidrs` - through the `iprange` command to merge/dedup and optimize the results.  This will output a single list of mixed single addresses, IP ranges (e.g. x.x.x.x-x.x.x.y) and CIDR ranges (e.g. 192.168.0.0/24) which do not contain overlaps (this is critical).

The `nftables.blockcidrs` list is the only non-mutable part of this chain.  It permits the user to specify a CIDR-specified list of ranges they *always* want blocked, whether known threats, GeoIP ranges, whatever.

When `nftblocklist` is invoked with the `-l` (load) option, it will first flush the contents of the set `blocklist_persistent` and then load the contents of the persistence file (again, by default, `/etc/nftables.blocklist`) into that set.

As examination of the conf file shows, one of the first rules present in the `inet filter` chain will DROP any traffic from ipv4 addresses present in either the `blocklist_persistent` or `blocklist_dyn` sets.  Thus, running `nftblocklist` with the -r (reload) flag at least once every 49 hours will ensure that any address that has been dynamically blocked will be 'promoted' to the persistent blocklist, as well as picking up any changes made to `nftables.blocklistcidrs`.

**NOTE:**  Although no operations performed by `nftblocklist` stop or restart the nftables service, there is a tiny interval during a *load* operation where the script first flushes `blocklist_persistent` before immediately loading in the contents of the blockfile.  This is necessary because any attempt to upload two entries with overlapping ipv4 will cause the `nft` tool to error out.

#### Timeouts and such
This is a pretty draconian blocklister.  Althought the `blocklist_dyn` set has a 49h timeout, the `blocklist_persistent` does not.  This is by design.  If, however, you wanted to be a bit more forgiving, you can certainly set a `timeout 14d` or similar flag on your `blocklist_persistent` in the conf file.  Be aware, though, that the setup current will keep appending the blocklists out of the nftables.blocklist *file on disk* - so it'll just get added back in next run of the script, because the 'load' operation flushes the elements including their timeouts and readds them.  If you want to maintain a timeout, it might make more sense to then modify the script to not cat the current blocklist, but instead just combine and optimize the manual `nftables.blockcidrs` file or equivalent, and the dump of the dynamic set.  This is hacky, but then again, so is all of this to some degree.  For more cheezwhiz, you could add the blockcidrs content into a `blocklist_perm` set and add a check for that, of course, so that 'persistent' is just the dynamics you want to preserve in between restarts.