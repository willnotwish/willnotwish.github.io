---
layout: post
title:  "Connecting Adhearsion to freeswitch via rayo"
date:   2016-11-08 10:15:00 +0000
categories: freeswitch rayo configure adhearsion
---

Following a fresh Freeswitch installation, here's what I did, following the [Adhearsion docs](http://adhearsion.com/docs/getting-started/freeswitch). I also got info from [here](https://github.com/chewi/freeswitch-rayo-config).

1. Started Freeswitch in the foreground with `sudo ./freeswitch -nc`
2. Connected to Freeswitch's CLI with `sudo fs_cli`
3. Freeswitch is running in the background on 192.168.0.14 (`fs`)
4. Set up a snom phone (`snom`) on 192.168.0.18
5. My iMac on 192.168.0.13 (`imac`)
6. My iPhone on 192.168.0.12 (`phone`)
7. Let's try running Adhearsion on my iMac, connecting to the [xmpp server provide by mod_rayo](https://freeswitch.org/stash/projects/FS/repos/freeswitch/browse/conf/rayo/autoload_configs/rayo.conf.xml). In `rayo.conf` I see 

```xml
    <!-- XMPP server domain -->
    <domain name="$${rayo_domain_name}" shared-secret="ClueCon">
    <!-- use this instead if you want secure XMPP client to server connections.  Put .crt and .key file in freeswitch/certs -->
    <!--domain name="$${rayo_domain_name}" shared-secret="ClueCon" cert="$${base_dir}/certs/$${rayo_domain_name}.crt" key="$${base_dir}/certs/$${rayo_domain_name}.key"-->

        <!-- Listeners for new Rayo client connections -->
        <!--listen type="c2s" port="5222" address="$${local_ip_v4}" acl="rayo-clients"/-->
        <listen type="c2s" port="5222" address="$${rayo_ip}" acl=""/>

        <!-- Listeners for new server connections -->
        <!--listen type="s2s" port="5269" address="$${local_ip_v4}" acl="rayo-servers"/-->

        <!-- servers to connect to -->
        <!--connect port="5269" address="node.example.com" domain="example.com"/-->

        <!-- Authorized users -->
        <users>
            <user name="usera" password="1"/>
        </users>
    </domain>
```

In `/usr/local/freeswitch/conf/vars.xml` I have `rayo_domain_name` set to `nickadams.co.uk` and `rayo_ip` to `127.0.0.1`

That means that the xmpp server is binding ti localhost, which won't work if I want to connect from another machine. I changed it to `0.0.0.0` and reloaded FS on the cli with `reloadxml`.

On `fs`, I did `netstat -an` but I could only see a listener on 127.0.0.1 on port 5222. So I restarted Freeswitch.

That worked, so I should be able to connect from my `mac`. I tested it with `telnet 192.168.0.14 5222`: it connected OK and when I typed some garbage I saw some log entries come up in `fs` in the cli.

8. I also registerd the `snom` using its web interface. A good command to check SIP registrations is `sofia status profile internal reg`. I don't really understand all this sofia profile stuff. I don't know why I need different profiles: I think I can manage with only one. I'll look at that later.
