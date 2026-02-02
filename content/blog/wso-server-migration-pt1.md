+++
title = "The WSO Server Migration, Part 1"
date = "2025-10-12T19:00:00Z"

#
# description is optional
#
# description = "What happens when you have to untangle 15 years of legacy nonsense? You learn a lot of lessons about how to be a sysadmin with future maintainers in mind."

tags = ["WSO",]
+++

# Introduction

First, a bit of context: [WSO](https://wso.williams.edu), officially *Williams Students Online*, is a project started in 1995 at Williams College to provide student online services that the college was either unwilling or unable to provide for the community (such as FacTrak, an early predecessor to services like "Rate My Professor"). Amazingly, it had been hosted on nearly the same hardware for about a decade now, and has mutated to the point where it is hardly recognizable as a classic LAMP (Linux, Apache, MySQL, PHP):

- We still use CentOS 6, but someone compiled a custom kernel and backported a substantial amount of mainline Fedora packages, resulting in a very strange `uname` and broken package updates. An upgrade to AlmaLinux directly from this setup would be a terrible idea.
- Apache is still there, but at some point a previous admin decided to run it as a submodule of cPanel, so now they are fused in a way where trying to kill cPanel kills Apache and vice versa. We don't use the cPanel interface, but effectively can't kill it because of what will happen to production.
- MySQL has been replaced with MariaDB, which might be the only good change over the last decade.
- PHP is still stuck on the same old version. This is extremely bad, but fortunately the PHP code on the site now is just MediaWiki, as we've rewritten all our own code in TypeScript on the frontend and Go on the backend.

This was the state that I found our production in. Rather than repair this mess, the faster solution was ultimately to go in and reimplement the entire thing. 

# Ansible Everything

One of the flaws of WSO is that our leadership turns over every four years (we graduate), which results in us perpetually having to re-explain ourselves to the future students who come after us. Fortunately, Ansible is actually the perfect tool for this job. By writing even a little bit of YAML, it becomes quite obvious what is supposed to be happening to future generations of students, even if Ansible becomes inoperable (unlikely as long as they're still blacked by Red Hat) or our configuration stops working (much more likely). Here's an example:

```
- name: Create WSO user
  ansible.builtin.user:
    name: wso
    comment: WSO Sysadmin Role
    uid: 1793 # year Williams was founded
    create_home: true
    shell: /bin/zsh
    generate_ssh_key: true
    ssh_key_type: "ed25519"
    ssh_key_comment: "Ansible-generated WSO Sysadmin key on $HOSTNAME"
```
This is much better than the alternative digital archaeology method:
```
$ cat /etc/group && cat /etc/passwd
...
$ group wso
...
$ ls -Alh /home/wso/
$ # and so on...
```
The only problem with Ansible, besides that it's slow, is that it's hard to train students on best development practices with Ansible. It is all too tempting to not test code in a VM first when setting up a VM can be quite annoying and take a while (we can't use containers because some of our code works directly with the running kernel and hardware in a way that Docker can't simulate).

Once all the Ansible code was re-written, and production was mostly stable we could finally move on to readjusting the fleet...

# The WSO Server Fleet

> There are two kinds of computer users: those who have yet to lose data and those who make backups. <br>
> — Anonymous

On the Williams College campus, there is a building called Jesup, and in its first floor server room is a singular server rack running WSO. This is the chokepoint for the entire service: if this machine goes down, all of WSO goes down with it. This is obviously alarming, doubly so since there isn't a backup.

What makes this arguably even more alarming is that this actually isn't what it was like back even in the early 2010s. Our hardware stack used to look like[^1]:
| Name | Specs | Purpose | Purchased |
|------|-------|---------|-----------|
|Wanda|PowerPC Mac mini|???|2005|
|Emma|PowerEdge R510, IDE Drive, 4GB RAM|Ancient production|2010|
|Yeti|PowerEdge R420, SATA Drive, 8GB RAM|Listserv host|2015|
|Sasquatch|PowerEdge R620, 32GB RAM|Old production|2013|
|VM-1|CentOS 6 in VMWare|Current production|2017|
|VM-2|CentOS 6 in VMWare|Current development|2017|

Astute readers may note the complete lack of a backup server anywhere on this list. Fortunately, a faculty member in the CS department was nice enough to give us a server they had lying around, although this was unfortunately not sitting in a server room somewhere but rather in some random lab on campus. This meant that silly things such as a student tripping on the power cord or the power lines failing during a particularly strong nor'easter could take out the backup server, which is obviously bad.

Running in a VM is fine and safer than running without one; plus, it makes it easy to migrate hardware. Thus, in our new hardware stack, all our machines run in VMs:
| Name | Specs | Purpose | Purchased |
|------|-------|---------|-----------|
|WSO-Prod|AlmaLinux 10, 6 cores, 16GB RAM|Future production|2025|
|WSO-Dev|AlmaLinux 10, 4 cores, 16GB RAM|Future development|2025|
|WSO-Stats|AlmaLinux 10, 2 cores, 5GB RAM|Future status host|2025|

# Benefits of Cleaning House

Having new hardware and a clean software stack lets us do many incredible things we couldn't possibly achieve before. One of the main benefits of this was that we can finally switch to `nginx` and automatically encrypt everything with HTTPS (yes, we still hadn't done it, `EasyApache` is just that awful to use). It also means that was once thousands of lines of Apache duct tape to force an HTTP to HTTPS redirect simplifies down to just:
```config
server {
    listen 80;
    listen [::]:80;    

    server_name wso.williams.edu;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    keepalive_timeout 70;

    server_name wso.williams.edu;
    ssl_certificate /etc/ssl/certs/wso_williams_edu.pem;
    ssl_certificate_key /etc/ssl/certs/wso_williams_edu.key;
     
    # rest of config, etc.
}
```
It also means that we can implement FoundryVTT and Minecraft, two hotly requested services from some clubs on campus. For Minecraft, we set up a basic Java server with a prerendered server map on a `cron` job, and thanks to having so much extra compute, it basically doesn't affect performance since it runs in the very early morning (4AM).

Overall it's quite fast and uses even less RAM than I had expected. Even with many users both gaming and on WSO at once, it struggles to make a real dent in the server's impact. Foundry was even easier with less than a couple dozen lines of `nginx` configuration to get working.

Our new hardware also came with a massive NFS server, which we use with `borg` to backup on a daily basis. This is much better than what cPanel was doing before (making a giant `.tar.gz` file and hoping for the best).

Another unexpected benefit is that server performance is much better. There are many legacy services running on old production, so the new machine is ~10% faster simply by lacking all the bloat, and it's easier to manage since there isn't nowhere near as much legacy cruft as there used to be. It also saves on disk space massively. There are tens of thousands of files on old production where we don't know why they're still there but can't get rid of them because some ancient service dies when they're moved, bringing the rest of the house of cards down with it. 

# Performance Implications

There are still many services left to port over:
- The core WSO service (a massive Go blob) itself 
- The Willipedia internal wiki
- Some of the special mobile/LDAP authentication features we use in our code

When these are done, we can finally retire old production. Until then, it'll be in this chimera state between new and old production; there will be another post when that migration happens, as it's sure to be interesting.

[^1]: If you're wondering why we didn't use any of the other ones as a backup, it's because we have to hand them back to the college once we migrated off of them.
