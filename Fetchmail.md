---
layout: doc
title: Fetchmail
permalink: /doc/Fetchmail/
redirect_from: /wiki/Fetchmail/
---

Fetchmail
=========

Fetchmail is standalone MRA (Mail Retrieval Agent) aka "IMAP/POP3 client". Its sole purpose is to fetch your messages and store it locally or feed to local MTA (Message Transfer Agent). It cannot "read" messages — for that use MUA like Thunderbird or [Mutt](/wiki/Mutt).

Installation
------------

`yum install fetchmail`

Configuration
-------------

Assuming you have more than one account (safe assumption these days), you need to spawn multiple fetchmail instances, one for each IMAP/POP3 server (though one instance can watch over several accounts on one server). The easiest way is to create template systemd unit and start it several times. Fedora does not supply any, so we have to write one anyway.

**NOTE:** this assumes you use [Postfix](/wiki/Postfix) as your local MTA.

In TemplateVM create `/etc/systemd/system/fetchmail@.service`:

{% highlight trac-wiki %}
[Unit]
Description=Mail Retrieval Agent
After=network.target
Requires=postfix.service

[Service]
User=user
ExecStart=/bin/fetchmail -f /usr/local/etc/fetchmail/%I.rc -d 60 -i /usr/local/etc/fetchmail/.%I.fetchids --pidfile /usr/local/etc/fetchmail/.%I.pid
RestartSec=1
{% endhighlight %}

Then shutdown TemplateVM, start AppVM and create directory `/usr/local/etc/fetchmail`. In it, create one `.rc` file for each instance of fetchmail, ie. `personal1.rc` and `personal2.rc`. Sample configuration file:

{% highlight trac-wiki %}
set syslog
set no bouncemail
#set daemon 600

poll mailserver1.com proto imap
    no dns
    uidl
    tracepolls
user woju pass supersecret
    ssl
    sslproto "TLS1"
    sslcertfile "/etc/pki/ca-trust/extracted/openssl/ca-bundle.trust.crt"
    sslcertck
    mda "/usr/sbin/sendmail -i -f %F -- user"
    fetchall
    idle

# vim: ft=fetchmail
{% endhighlight %}

Then `chown -R user:user /usr/local/etc/fetchmail` and `chmod 600 /usr/local/etc/fetchmail/*.rc`. **This is important**, fetchmail will refuse to run with wrong permissions on its rc-file.

Next, add this to `/rw/config/rc.local`:

{% highlight trac-wiki %}
#!/bin/sh

for rc in /usr/local/etc/fetchmail/*.rc; do
        instance=${rc%.*}
        instance=${instance##*/}
        echo systemctl --no-block start fetchmail@${instance}
done
{% endhighlight %}

Now reboot your AppVM and you are done.
