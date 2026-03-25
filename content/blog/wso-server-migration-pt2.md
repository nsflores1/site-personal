+++
title = "The WSO Server Migration, Part 2"
date = "2026-02-01T23:41:25-05:00"

#
# description is optional
#
# description = "How the server transfer was completed"

tags = ["wso",]
+++

# Previously on WSO...

The big challenge last time that kept me from finally moving everything over to the new server was LISTSERV. This ancient piece of email software dates back to the Paleolithic[^1], and is a maintenance nightmare precisely because it's annoying to get data out of.

Adding to this burden was cPanel, another nasty fossil that needed to be relegated to the past. Unfortunately cPanel also had all of the configuration for `exim` and LISTSERV in it, so the big task would be figuring out how to extract it from the running server in a way that is safe, and then reinstalling as much of the mail archives and list data as possible onto the new server.

How hard could it be?

# Challenges

There turned out to be no nice tool for converting LISTSERV files over to the Mailman 3 equivalents. Fortunately the cPanel install shipped with a script for converting LISTSERV contents to Mailman 2 files, and Mailman 3 ships with a script called `import23`, so I decided it would be most logical to chain them.

I decided to keep my sanity, and after several attempts to compile Mailman from source, I decided to use the Docker container for it instead. (If you're reading this and somehow figured it out: please email me! I don't like running it in Docker and want to get it out of there, but the Python build process is so arcane that it simply won't build on my RHEL install.)

The trick is simple: loop over the files with a little Bash glue. I'm not copying it here because if you're doing this, you should really just use the Python interface. Unfortunately, the Mailman 3 Python interface is so bad that you'll need to `git clone` their source code and `grep` through it yourself to have any hope of finding the APIs you seek.

The other main problem was convincing Williams College to open up port 25 for us, but once that was done everything was quickly in working order again. Our server could receive mail! (Fortunately `rspamd` can take in Bayesian statistics from `exim`'s mail data, so setting it up as a milter for automatic spam detection was relatively easy. I recommend this, `rspamd` is honestly the least painful part of the mail stack, although connecting it to `postfix` can be. Make sure to drop mail with `spamhaus` at SMTP time and `rspamd` will be nice and idle 90% of the time.) 

# Web Upgrades

Moving from an ancient version of Apache to modern NGINX was quite nice, actually. Much nicer than I had anticipated: we gained so much server performance from a finely-tuned NGINX for our needs compared to whatever cPanel was doing with its embedded Apache that the server idling and thousands of users browsing the (mostly-static) WSO website are basically identical from a load perspective. Goes to show how much better web servers have gotten since 2008 (the time from which our install of Apache dates to).

One unexpected issue is that swapping to HTTP/2 resulted in many subtle application issues. I'm amazed that this is still a problem, but I ended up needed to edit and recompile our backend and frontend code to handle the fact that `200 OK` is no longer sent, but instead just the raw numerical code `200`. This should be commonplace knowledge in 2026, but shockingly this assumption was everywhere in our code. The longevity of HTTP 1.1 support has resulted in some cursed code. 

This also gave time to upgrade the security of the site, and to add some hardening headers to prevent XSS attacks and to improve caching performance. Much of the performance gains made have been from actual caching on the NGINX side; not having to constantly reload the same 20MB JSON from disk saves a lot of time when thousands of users are requesting it at once.




[^1]: Not really, but our install dated back to 1991 (the start of WSO!), so it might as well be in terms of computer time. Nearly 30 years unmodified, still running on some ancient Perl version is quite the feat.
