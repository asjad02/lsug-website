# Table of Contents

1.  [Overview](#org79d0c7e)
2.  [EC2 Instance configuration](#org47ece93)
3.  [The SSL Certificate](#org4373427)
    1.  [Certbot initial setup](#org0cf11fe)


<a id="org79d0c7e"></a>

# Overview

We use an AWS cloud architecture to run the website. Cloud
architectures usually involve multiple machines and environments,
hence have a complicated deployment.

The LSUG website is much simpler. There’s only one environment —
production — with a single machine. We don’t even have a load
balancer.

This has some minor downsides.

Any changes to the deployment pipeline must be tested in production,
and may bring the site down. Additionally, the site is taken down for
a brief moment whenever it’s re-deployed. Since we have a fully
automated deployment pipeline, it could be taken down fairly often.

The machine is an AWS EC2 instance.


<a id="org47ece93"></a>

# EC2 Instance configuration

Large architectures generally need tools to manage machine
configuration (puppet or ansible, for example). We have a single host
running a single web server, so don’t need these.

We configure the instance directly through the [cloud-init User-Data](https://cloudinit.readthedocs.io/en/latest/topics/format.html)
script. This contains some yaml and a shell script.

The shell script is run on boot and does the following:

-   It downloads the jar and assets
-   It downloads the SSL certificate and certificate renewal files
-   It starts a timer for certificate renewal
-   Finally, it starts the lsug service

The User-Data is coded as a string in `stack.sc`. There are strings
within strings, as well as both bash and Scala variable substitutions,
so wrangling this can be frustrating.


<a id="org4373427"></a>

# The SSL Certificate

The SSL certificate is stored in S3 in a PKCS12 file. It is downloaded
during boot and its path and password are handed to the web server as
arguments.

The certificate needs to be renewed every few months. This task is
handled automatically by [certbot](https://certbot.eff.org/docs/using.html).

Certbot is configured in the User-Data script. It is installed on
instance creation, and comes with the `certbot-renew` systemctl service
and timer to automate it.

It needs a few files to be set up in the `/etc/letsencrypt/`
directory. These were generated when certbot got the original
certificate. The first run was manual, with a maintainer hopping onto
the machine to run certbot commands.

As described in [the user guide](https://certbot.eff.org/docs/using.html#where-are-my-certificates), Certbot needs the following files in
`/etc/letsencrypt/archive/www.lsug.co.uk/`:

-   `cert.pem`
-   `chain.pem`
-   `fullchain.pem`
-   `privkey.pem`

These are downloaded from S3 on boot, placed in
`/etc/letsencrypt/archive/www.lsug.co.uk/` and are symlinked to
`/etc/letsencrypt/live/www.lsug.co.uk`.

It also needs a `/etc/letsencrypt/renewal/www.lsug.co.uk.conf` file to
tell it where the other files are, and to give a bit more detail on
domains.

Certbot temporarily stops the web server while it renews
certificates. This is done using [pre and post hooks](https://certbot.eff.org/docs/using.html?highlight=renewing%20certificates#renewing-certificates).

Once it completes, it symlinks the new certificate in
`/etc/letsencrypt/live/www.lsug.co.uk`. A PKCS12 is generated from this
and uploaded to S3, along with the other pem files. Finally, the lsug
service is restarted to pick up on the new certificate. This is
configured in a post-deploy hook `/usr/local/bin/lsug-cert`.

The certbot systemctl timer runs is scheduled to run once per day, at
midnight. No-one will be awake to see the site go down.

You can inspect the service and timer by hopping onto the host and
using the usual systemctl commands.

    systemctl list-timers certbot-renew.timer

    NEXT                         LEFT     LAST                         PASSED       UNIT                ACTIVATES
    Fri 2021-01-15 09:51:17 UTC  12h left Thu 2021-01-14 18:11:40 UTC  3h 18min ago certbot-renew.timer certbot-renew.service

    1 timers listed.

The exact certbot command can be found in the `certbot-renew.service`
file. It can be run with a `--dry-run` flag to troubleshoot.


<a id="org0cf11fe"></a>

## Certbot initial setup

The original certificate was obtained manually. It was obtained by:

-   Installing Certbot using `yum` from the [EPEL](https://fedoraproject.org/wiki/EPEL#What_is_Extra_Packages_for_Enterprise_Linux_.28or_EPEL.29.3F) repository.
    The installation configuration can be found in the User-Data
    script.

-   Terminating the web server to free up port 80

-   Obtain the certificate for `www.lsug.co.uk` and `www.lsug.org`

This generated the required files in the
`/etc/letsencrypt/live/www.lsug.co.uk/` directory.

If you ever need to obtain a new certificate, consult the [certbot docs](https://certbot.eff.org/docs/using.html#standalone)
instead of these steps.
