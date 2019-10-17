---
layout: post
title:  "My Pi-Hole Setup"
---

I started using Pi-hole about a year ago. I was browsing the [/r/raspberry_pi](https://www.reddit.com/r/raspberry_pi/) subreddit for ideas for a project that I could do with my newly-acquired Raspberry Pi 3B+, and after skimming through a few questions and tutorials, I stumbled upon [Pi-hole](https://pi-hole.net/). For those unfamiliar with the Pi-hole project: Pi-hole is an open-source DNS sinkhole that can block adverts for *all* devices on your network. This sounded great, so I set about installing it straight away.

Since then, my Pi-hole has moved onto a different Raspberry Pi (it now runs on my Raspberry Pi Zero W) and has been running pretty much continuously. After some tinkering, I have tweaked the setup to a point where I am reasonably happy with it, so the purpose of this blog post is to document some of those tweaks. This is partly for my own benefit (so that I have a place to look when I inevitably forget some of the steps) but I hope that this will also prove useful for others who are looking to achieve some of the same things.

#### Contents
{:.no_toc}

* 
{:toc}

### Hold on - I don't have a Pi-hole!

If you're new to Pi-hole then check out the installation guide [here](https://docs.pi-hole.net/main/basic-install/). For blocklists, I use the 'ticked lists' from [this site](https://v.firebog.net/hosts/lists.php) and so far I haven't encountered any troublesome false positives.

If you've got a Pi-hole, then read on...

### Using a custom domain for the Pi-hole

The first tweak that I wanted to make was to use my own domain name for the Pi-hole. I have a few different devices on my LAN, and there's something quite satisfying about having them all use a nice domain name with a real TLD. Also, as we'll see later, using a real domain name makes the SSL setup a lot nicer.

#### Step 0: Get a domain name

Before going any further, you're going to need a domain to use, so go ahead and register one if you haven't already. Note: if you're *not* planning on securing the admin interface using SSL then make sure you don't use a TLD that is on the [HSTS](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) preload list (_e.g._ don't use ```.dev```). If you do use one of those TLDs then you might not be able to access the admin interface using your domain until you've completed the SSL set up below as well.

My internal domain name is ```mattbattison.net``` and my pi-hole is accessed at ```pihole.mattbattison.net```, so those are the names I'm going to use in the instructions below.

#### Step 1: Adding a custom DNS entry for the Pi-hole

The first step is to make sure that your custom hostname will be resolved locally to the IP address for the Pi-hole. In order to do this, you'll need to add a custom DNS entry to your local DNS server. Fortunately, the Pi-Hole *is* your local DNS server, so this is relatively straightforward.

The configuration we need to edit is in the ```/etc/dnsmasq.d/``` directory (dnsmasq is the DNS forwarder on which Pi-hole's DNS engine is based). In there, you'll probably find a couple of ```.conf``` files:

```
01-pihole.conf
04-pihole-static-dhcp.conf
```

Let's add a new one called ```09-pihole-custom.conf``` with the following content:&nbsp;[^dnsmasq-man]

```
# Additonal custom DNS entries
addn-hosts=/etc/pihole/custom.list
```
[^dnsmasq-man]:<http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html>

What we're doing here is saying to dnsmasq that it should look in the file ```/etc/pihole/custom.list``` when attempting to resolve a hostname and that it should use the entry in there if it exists rather than forwarding the request out to an upstream server.

So now we need to actually create that file. Navigate to ```/etc/pihole``` and add a new file called ```custom.list``` with the following content:

```
192.168.1.10 pihole.mattbattison.net
```

replacing ```192.168.1.10``` with the IP address of your Pi-hole, and ```pihole.mattbattison.net``` with the hostname you want to resolve to your Pi-hole.

After you've done the above, run ```pihole restartdns``` on your Pi-hole. Then, from a *different* device on the network, run ```host pihole.mattbattison.net``` (again, replacing my hostname with yours). If everything's configured correctly then you should get back the IP address of your Pi-hole.

#### Step 2: Allowing access to the admin interface

Now we've pointed the domain at the Pi-hole, we should be able to navigate to the Pi-hole splash page using that domain, right? Unfortunately, if you try going to your newly-configured hostname in the browser, you'll probably see something like the following:


```
Access to the following website has been denied:

    pihole.mattbattison.net
```

What's going on? Well, by default, when Pi-hole blocks a domain, it gives out its own IP address instead of the 'real' IP address for that domain. So when it receives a request for a host that it doesn't recognise, it assumes it got there because it was a blocked domain.

We need to tell Pi-hole that for *our* domain, it should show the splash page instead.

To do this, we need to edit the configuration for the webserver that the Pi-hole uses to serve the admin interface. Specifically, we need to edit (or create if it doesn't exist) ```/etc/lighttpd/external.conf```, adding the following content:&nbsp;[^fqdn-config]

```
# Don't consider pihole.mattbattison.net as a blocked domain
$HTTP["host"] == "pihole.mattbattison.net" {
    setenv.add-environment = ("fqdn" => "true")
}
```
[^fqdn-config]: <https://github.com/pi-hole/pi-hole/blob/e41c4b5bb691cea1f5b950d39518d8c404b5846e/advanced/index.php#L29>

After you've saved this, restart the webserver by running ```sudo systemctl restart lighttpd```. You should now be able to access the splash page and admin interface using your new hostname!

#### Step 3: Redirecting ```pi.hole``` to your hostname

You'll notice that it's still possible to access to the admin interface by navigating to ```pi.hole/admin```. This is fine, but it would be nice if it redirected us to the new hostname so that we're always accessing it at the same address. This is especially useful if, for example, you have saved the admin password in your browser against the new hostname.

To set up the redirection we need to change the lighttpd config again, so head back to ```/etc/lighttpd/external.conf``` and add the following redirect rule:&nbsp;[^lighttpd-redirect]

```
# Redirect pi.hole to pihole.mattbattison.net
$HTTP["host"] == "pi.hole" {
    url.redirect = ( "^/(.*)" => "http://pihole.mattbattison.net/$1" )
}
```
[^lighttpd-redirect]:<https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_ModRedirect>

After you restart lighttpd again (```sudo systemctl restart lighttpd```) then you should be redirected when you navigate to ```pi.hole```.

If your Pi-hole is accessible via any other hostnames (for example if its *actual* hostname is different from the new one you have configured -- run the ```hostname``` command to check) then you may also want to set up a similar redirect rule from those hostnames.

You've now successfully set up your Pi-hole to use a custom domain!

### Securing the admin interface using HTTPS

Once I had set up the Pi-hole to use my ```mattbattison.net``` domain, I realised that it should be possible for me to obtain an SSL certificate, allowing me to serve the admin interface over HTTPS and to avoid sending the admin password in the clear.

Technically, this is possible with the ```pi.hole``` domain too, but it requires a self-signed certificate, which ends up presenting a rather scary error message to anyone who attempts to navigate to the interface. We don't want that.

#### Step 1: Obtain an SSL certificate

The first step is to get hold of an SSL certificate. There are a number of ways to do this, and I'm not going to go into great detail; there are plenty of tutorials out there. If you're unsure how to proceed, I suggest checking out [Let's Encrypt](https://letsencrypt.org). You can use their ```certbot``` tool to obtain a free SSL certificate.

If you are using certbot, bear in mind that your Pi-hole probably (hopefully?) *isn't* accessible from the internet, so you'll have to run it in manual mode and use the DNS challenge. For example:

```
$ certbot certonly --manual --preferred-challenges dns -d pihole.mattbattison.net
```

This will ask you to place specific TXT entries in the public DNS records for your domain as a means of verifying that you have control over the domain.

Once you've obtained a certificate, you should have some combination of the following files (potentially with different names -- the names below are those used by Let's Encrypt):

- the certificate (```cert.pem```)
- the full chain of certificates (```fullchain.pem```)
- the private key (```privkey.pem```)

In the next steps I'm going to assume that your files have the names shown above. If they're different then you'll have to tweak the commands to match.

#### Step 2: Moving things into place

To configure lighttpd, the web server used for the Pi-hole admin interface, we need two files: one containing the private key & certificate and the other containing the full chain.[^lighttpd-ssl] Let's combine ```privkey.pem``` and ```cert.pem``` into a single file called ```combined.pem```:
```
$ cat privkey.pem cert.pem > combined.pem
```
[^lighttpd-ssl]:<https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_SSL#Configuration>

Then we can move the files we need into a new directory inside the lighttpd configuration directory:

```
$ mkdir /etc/lighttpd/ssl
$ cp combined.pem fullchain.pem /etc/lighttpd/ssl
```

Finally, make sure these files are readable by the lighttpd user (```www-data``` by default), either by changing the owner:
```
$ chown www-data: /etc/lighttpd/ssl/*
```
or by making them readable by all (don't do this if you're worried about the private key being read by other programs):
```
$ chmod +r /etc/lighttpd/ssl/*
```

#### Step 3: Configuring lighttpd to use HTTPS

Now that we have the certificate files in a place where lighttpd can see them, we can edit the lighttpd configuration. Once again, the file to edit is ```/etc/lighttpd/external.conf```. We need to add some extra lines inside the block for our Pi-hole's hostname:
```
$HTTP["host"] == "pihole.mattbattison.net" {

    ...

    # Enable SSL
    $SERVER["socket"] == ":443" {
        ssl.engine = "enable"
        ssl.pemfile = "/etc/lighttpd/ssl/combined.pem"
        ssl.ca-file = "/etc/lighttpd/ssl/fullchain.pem"
    }

}
```

This tells the server to enable SSL on port 443 (default HTTPS port) and to use the certificate files we just moved into place. This is the minimal config; there are loads more options you can set if you want to be more secure (_e.g._ you can specify which cipers and SSL versions to allow). See [this FAQ on the Pi-hole community](https://discourse.pi-hole.net/t/enabling-https-for-your-pi-hole-web-interface/5771) for a more complete example.

If you restart lighttpd now (```sudo systemctl restart lighttpd```) then you should be able to navigate to the admin interface using HTTPS. If all is working correctly then you'll see a nice little padlock in your browser's address bar!

#### Step 4: Redirecting HTTP to HTTPS

Finally, to enforce use of the secure protocol, we should redirect any HTTP requests to use HTTPS. This requires just a few more lines in the same block in ```/etc/lighttpd/external.conf```:&nbsp;[^lighttpd-redirect]
```
$HTTP["host"] == "pihole.mattbattison.net" {

    ...

    # Redirect HTTP -> HTTPS
    $HTTP["scheme"] == "http" {
        $HTTP["host"] =~ ".*" {
            url.redirect = (".*" => "https://%0$0")
        }
    }

}
```

Now if someone tries to go to ```http://pihole.mattbattison.net/``` they'll be redirected to ```https://pihole.mattbattison.net/```. Your Pi-hole admin interface is secure!

_Note: if you obtained your certificate from Let's Encrypt then it's likely that it will expire after 90 days. You might want to set up a script (or at least a reminder) to renew your certificate as it nears expiry._

### Resolving arbitrary sub-domains to a particular device

The final thing that I wanted to set up was a catch-all DNS rule that would route all requests for ```*.pi.mattbattison.net``` to my other Raspberry Pi, which I am using as a local web/CI server. Having a catch-all rule is useful because it means I can host local staging versions of sites using unique hostnames (e.g. ```mattbattison.dev.pi.mattbattison.net```) without having to manually add DNS rules for each one.

The configuration for this is actually very simple. We just need to add a single line at the top of the ```/etc/dnsmasq.d/09-pihole-custom.conf``` file that we created earlier:&nbsp;[^dnsmasq-man]
```
# Wildcard entry for *.pi.mattbattison.net
address=/.pi.mattbattison.net/192.168.1.20
```

Obviously, you'll need to replace the name and address with the corresponding values for your set-up.

You should then be able to navigate to ```x.y.x.pi.mattbattison.net``` and it will redirect to the given IP. You'll still have to set up the virtual hosts on the receiving web server, though.

# Conclusion

I hope these instructions proved useful for configuring your Pi-hole. If nothing else, they'll be here to remind me how I did this in six months time when I've accidentally wiped my Pi and need to set it up all over again.

---

