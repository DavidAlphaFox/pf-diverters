# pf diverters

A collection of daemons written for [OpenBSD](http://www.openbsd.org/) 
[PF](http://www.openbsd.org/faq/pf/), that listen on 
[divert(4)](http://www.openbsd.org/cgi-bin/man.cgi?query=divert&sektion=4) 
sockets.

PF can be configured to send matching packets to a divert socket via the 
parameter `divert-packet port <port>`. Divert sockets are bound to divert ports
(completely separated from tcp/udp) and enable us to queue raw packets from the
kernel stack to userspace applications and vice versa. 

This synergy leaves plenty of space for innovation; matching packets from PF 
can be stopped from propagating through the IP stack, in order to be brought to
our userspace daemons, and optionally be re-injected back into the kernel stack
for normal processing. Certainly, the daemons can perform additional checks on
intercepted connections and, based on those checks, immediately enforce 
firewall policy.

```
WARNING: THESE TOOLS ARE EXPERIMENTAL AND IN NO-WAY PRODUCTION READY.

Feel free to test and run them on your systems, but make sure you keep a close eye.
```

List of diverters available:
  
* `bofh-divert` Divert connections to this daemon and add each src host to a 
  predefined PF table (used for banning abusers). 
* `dnsbl-divert` Divert connections to this daemon and check if the source ip 
  is on a dnsbl and drop packet, or else reinject packet to reach its original
  destination. (still work-in-progress)

## Building

On an OpenBSD system, get the source and simply run make:

<sub>Note: if git(1) is not installed on your system, you can always download 
the code as a tar.gz archive ([http link](https://github.com/echothrust/pf-diverters/archive/master.tar.gz)).</sub>

```
$ git clone https://github.com/echothrust/pf-diverters
$ cd pf-diverters
$ make
```

This will compile the binaries for the diverters. If you wish, you can also run 
`make install` to place the executables in `/usr/local/sbin` and the rc scripts
in /etc/rc.d.

## Running

### bofh-divert

A simple divert socket daemon that can used to automaticaly block connections. 
With the help of PF, you redirect a bunch of unused (by you) ports to this 
daemon listening on a divert socket and hosts that attempt access are instantly 
added to a predefined PF table. Combined with a block rule for that table, this 
essentially sets tripwires for any attackers probing those unused TCP ports, 
effectively blocking the rest of the attempts originating from the same IP 
addresses.

```
$ ./bofh-divert                                                            
usage: bofh-divert <divert_port> <pf_table_name>
        <divert_port> divert port number to bind (1-65535)
        <pf_table_name> table to add collected host IPs (up to 32 chars)
```

Say you run `bofh-divert 1100 bastards`, you would also need the corresponding 
PF rules for this to work, in `/etc/pf.conf`, say for a list of well-known 
scanner ports:

```
table <bastards> persist counters
block in log quick from <bastards>
pass in log quick on { egress } inet proto tcp from !<bastards> to port { 22, 23, 81, 138, 139, 445, 1024, 3389, 5900, 6001, 8080, 8888 } divert-packet port 1100 no state
```

All daemon activity is appropriately logged through syslog, e.g.:

```
Sep 17 18:56:16 fw01 bofh-divert: attacker_ip:13477 -> your_ip:3389
```

### dnsbl-divert

A similar daemon that can be used on firewalls to fence connections on 
listening (used) ports. Based on DNS blacklists, source IPs can be validated 
prior to allowing the connection to happen.

```
$ ./dnsbl-divert                                                           
usage: dnsbl-divert <divert_port> <pf_table_black> <pf_table_cache> [dns_ip]
        <divert_port>    divert port number to bind (1-65535)
        <pf_table_black> table to populate with DNSBLed hosts (up to 32 chars)
        <pf_table_cache> table to cache already-looked-up hosts (up to 32 chars)
        <dns_ip>         DNS server address (default: use system-configured dns)
```

This is BETA/untested software that can take numerous improvements. Usage is 
very similar to bofh-divert, but this is destined for application in front of 
listening ports. For up-to-date running instructions, PF config and also for 
setting your prefered DNSBLs, please take a look in the source code.

### rc.scripts

Run controlscripts can be used to start diverters on system boot, for example:

```
ln -s /etc/rc.d/rc.bofh /etc/rc.d/bofh_bastards
echo 'bofh_bastards_flags="1100 bastards"' >> /etc/rc.conf.local
```

This will configure the system to start `bofh-divert` daemon on boot, listening 
on divert_port '1100' and logging offenders in PF table 'bastards'. Of course, 
PF should be configured to create the table 'bastards' and forward offending 
connections to divert_port 1100.

## Notes
  
The code is destined to compile and run on OpenBSD 5.3 amd64. It could also be 
suitable for other platforms featuring PF, but modifications may be needed. 

On OpenBSD, superuser privileges are required to open a divert socket (and thus
run these programs).

When dealing with pf tables you also need write access to /dev/pf.

All the diverters require the pre-existance of the pf table.

## Contributing

There sure is room for improvement, but also many ideas on similar diverters to
implement. Code contributions are always welcome:

1. [Fork it](https://github.com/echothrust/pf-diverters/fork)
2. Clone your forked project (`git clone https://github.com/YOUR-ACCOUNT/pf-diverters`)
3. (Optional) Create your feature branch (`git checkout -b my-new-feature`)
4. Add code as you see fit (introduce new files with `git add my-new-feature.c`)
5. Commit your changes (if any) to existing code (`git commit -am 'Add some feature'`)
6. Push back to your forked clone (`git push` or `git push origin my-new-feature`)
7. Create new Pull Request [![Info](https://help.github.com/assets/help/info-icon-ba11a61a3770bbc50703f444d80e915b.png "Creating a new Pull Request")](https://help.github.com/articles/creating-a-pull-request)
