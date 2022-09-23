---
title: Home Lab from First Principle
date: 2022-09-23 10:00:00 -000
categories: [homelab]
tags: [pxelinux]
---

# Home Lab from First Principle

I find the first principles in math quite fascinating... its the answer to the most important question, "chicken or the egg"??

So I find myself with a bare, but typical 'home network', an ISP router and some personal devices on the network. Then I added a pihole for DHCP and DNS. I needed some compute and was inspired by TiniMiniMicro (its about the price of raspberry pies, at the scalping prices, which noone should pay..)

[![image](https://www.servethehome.com/wp-content/uploads/2020/07/Project-MiniMicro-Cover-Introduction-800x450-Cover.jpg)](https://www.servethehome.com/introducing-project-tinyminimicro-home-lab-revolution/)

So.. how _does_ one bootstrap a cluster? I have not decided (or rather, 'learnt') how to properly arrange my homelab, would like to go back and adjust early decisions as I learn how wrong I am today. Or when add more 'things' to the lab.. **What is the 'first principle' for my cluster**, how can I bootstrap a computer without any other computer?

[MAAS](https://chris-sanders.github.io/2018-02-02-maas-for-the-home/) sounds ideal... but.. its certainly not 'small'. I need to bootstrap my lab, and MAAS wants postgress, user management.. way too much for home use. 

Then I stumbled on [How to configure a Raspberry Pi as a PXE boot server](https://linuxconfig.org/how-to-configure-a-raspberry-pi-as-a-pxe-boot-server)
That is much closer to what I need! pihole already has installed dnsmasq, so its even 'less work'! (as-if 😂)

So after some tinkering, yes, I can remote-boot a machine (what a backdoor... lets ignore that... need to get pfsense probably). YES!! Still. Problem. ubuntu install is not completely 'headless'. This is however an old problem. The solution.. keeps changing. Here is the best [post](https://askubuntu.com/questions/1235723/automated-20-04-server-installation-using-pxe-and-live-server-image) I found on the subject.

Now I have all the pieces, how to glue that all together. Just need 
- a `pxelinux.cfg` and `user-data` per machine
- some static content (ubuntu iso, `vmlinuz`, `initrd`, `system.efi`...)
- tftp and http fileserver to serve the above content
- dnsmasq to direct boot sequence to above http/tftp server
dnsmasq has a tftp server, http fileserver is a python one-liner, but.. copy-paste duplication of files is 'blasphemy'!! 

'Luckily', I spent several years working with go `text/template` (i.e. {% raw %}`{{.Varname}}`{% endraw %} syntax, it shows in a lot of tools). Go also has a file server.. and.. yes, there is an implementation of tftp! https://github.com/pin/tftp. So.. we can implement a bootstrap server in go!!!

Few hours later:

{% raw %}
```yaml
serve:
  host: <raspberry pihole hostname>
  http: 8080
  tftp: 69
  datadir: /static
inventory:
  12:34:56:78:9A:B3:  firefly1   
  12:34:56:78:9A:BB:  firefly2   
  12:34:56:78:9A:BE:  firefly3   
  12:34:56:78:9A:B5:  firefly4   
  12:34:56:78:9A:B7:  firefly5
passwordhash: # `apt-get install whois && mkpasswd <password>`
bootmenu: |
  DEFAULT install
  LABEL install
      KERNEL http://{{.Server}}:{{.HttpPort}}/static/efi64/vmlinuz
      INITRD http://{{.Server}}:{{.HttpPort}}/static/efi64/initrd
      APPEND root=/dev/ram0 ramdisk_size=1500000 ip=dhcp url=http://{{.Server}}:{{.HttpPort}}/static/ubuntu-22.04.1-live-server-amd64.iso autoinstall ds=nocloud-net;s=http://{{.Server}}:{{.HttpPort}}/autoinstall/{{.Hostname}}/ cloud-config-url=/dev/null
cloudinit: |
  #cloud-config
  autoinstall:
    updates: security
    version: 1
    reporting:
      hook:
        type: webhook
        endpoint: http://{{.Server}}:{{.HttpPort}}/cloudlog/{{.Hostname}}
    apt:
      disable_components: []
      geoip: true
      preserve_sources_list: false
      primary:
      - arches:
        - amd64
        - i386
        uri: http://{{.Server}}:3142/ubuntu
      - arches:
        - default
        uri: http://{{.Server}}:3142/ubuntu
    drivers:
      install: false
    identity:
      hostname: {{.Hostname}}
      password: {{.PasswordHash}}
      realname: vpaprots
      username: vpaprots
    kernel:
      package: linux-generic
...
```
{% endraw %}

Added an `apt-cacher-ng` on the pi (probably not a good solution long-term..) and call-home log reporting to the golang server. Then realized that I need to send WakeOnLine packets, added that too.. Since have a webserver, hooked WOL up to a GET request.. And for good measure, I thought it might be nice to see what part of the install we are on, which can be gleaned from what files have been requested so far! (First time using websockets.. and accidentally stumbled on `bootstrap.js`.. so many pieces!!) 

And.. voila.. (and I need to find how to make better gifs 😅)
![image](../../assets/images/pxeboot.gif)

Full code at [@vpaprots/pxeserver](https://github.com/vpaprots/pxeserver).