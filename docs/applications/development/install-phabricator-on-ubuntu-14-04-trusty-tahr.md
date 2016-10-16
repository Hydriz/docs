---
author:
    name: Linode Community
    email: docs@linode.com
description: 'Install Phabricator on Ubuntu 14.04 (Trusty Tahr).'
keywords: 'phabricator,git,svn,mercurial,mysql,php,apache,bug tracker,tracker'
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
published: 'Sunday, October 16, 2016'
modified: Sunday, October 16, 2016
modified_by:
    name: Hydriz Scholz
title: 'Install Phabricator on Ubuntu 14.04 (Trusty Tahr)'
contributor:
    name: Hydriz Scholz
    link: https://github.com/Hydriz
external_resources:
 - '[Phabricator](https://secure.phabricator.com)'
 - '[Phabricator Installation Guide](https://secure.phabricator.com/book/phabricator/article/installation_guide/)'
---

*This is a Linode Community guide. Write for us and earn $250 per published guide.*
<hr>

Phabricator is a software development platform with various tools and applications available as part of the suite. The tools include task management, code review and much more. Most importantly, Phabricator features code hosting in Git, SVN and even Mercurial, and provides the ability to mirror repositories hosted elsewhere.

Phabricator provides a tight integration among the different tools, allowing items from a certain application to be easily associated with another item on another application. Many organizations utilize Phabricator for software development, with the most notable one being Facebook.

This guide will walk you through the various steps of installing and setting up a Phabricator instance on a Ubuntu 14.04 (Trusty Tahr) Linode instance. If you are unfamiliar with how to administer a Linux system, you can consult either the [Introduction to Linux Concepts guide](/docs/tools-reference/introduction-to-linux-concepts) or the [Linux Administration Basics guide](/docs/tools-reference/linux-system-administration-basics).

## Before you begin

1. Familiarize yourself with our [Getting Started](/docs/getting-started) guide and complete the steps for setting your Linode's hostname and timezone.

2. This guide will use `sudo` wherever possible. Complete the sections of our [Securing Your Server](/docs/security/securing-your-server) to create a standard user account, harden SSH access and remove unnecessary network services.

3. Ensure that your system is up-to-date.

        sudo apt-get update && sudo apt-get upgrade

{: .note}
>
> This guide is written for a non-root user. Commands that require elevated privileges are prefixed with `sudo`. If you’re not familiar with the `sudo` command, you can check our [Users and Groups](/docs/tools-reference/linux-users-and-groups) guide.

## System requirements

It is recommended that Phabricator is installed on a machine with at least **2 GB of memory**. As Phabricator is extremely configurable, it is possible for Phabricator to run on a machine with a smaller memory space, but 2 GB of memory strikes a good balance between costs and performance.

The number of CPU cores depends on the amount of traffic that your Phabricator instance is likely to receive. More cores would be useful if your Phabricator instance receives a large amount of traffic.

At the time of writing, the Linode 2GB instance is sufficient for Phabricator's needs.

## Installation

Phabricator operates on the basic LAMP stack and Phabricator itself has provided a [simple installation script](https://secure.phabricator.com/diffusion/P/browse/master/scripts/install/install_ubuntu.sh) for most installation cases. Nonetheless, this guide will walk you through the individual steps for installing Phabricator.

{: .note}
>
> Phabricator currently only supports PHP 5 and PHP 7 is **not supported**.

### Install Package Dependencies

In this section, you will install the required components for Phabricator to function normally.

1. Install the packages required for setting up a webserver:

        sudo apt-get install mysql-server apache2 dpkg-dev php5 php5-mysql php5-gd php5-dev php5-curl php-apc php5-cli php5-json

2. Install Git:

        sudo apt-get install git

3. Install the PCTNL extension:

        apt-get source php5
        PHP5=`ls -1F | grep '^php5-.*/$'`
        (cd $PHP5/ext/pcntl && phpize && ./configure && make && sudo make install)

### Downloading Phabricator source

In this section, you will download the Phabricator source along with its required libraries. We will install Phabricator into `/mnt`.

1. Transfer the ownership of the `/mnt` directory to yourself and the `www-data` user (change $USER to your own username):

        sudo chown -R $USER:www-data /mnt

2. Change the working directory to `/mnt`:

        cd /mnt

3. Download the Phabricator source:

        git clone https://github.com/phacility/phabricator.git
        cd phabricator
        git pull --rebase
        cd ..

4. Download the other required libraries:

        git clone https://github.com/phacility/libphutil.git
        cd libphutil
        git pull --rebase
        cd ..
        git clone https://github.com/phacility/arcanist.git
        cd arcanist
        git pull --rebase
        cd ..

If the source code is successfully downloaded, you will be able to see the `phabricator/`, `libphutil/` and `arcanist/` libraries in the `/mnt` directory.

{: .caution}
>
> If you are setting up a production Phabricator instance, you should use the `stable` branch for each of the 3 repositories. Run `git checkout stable` in each of the 3 directories to do so.

## Configuration

Now that you have Phabricator installed and the required dependencies installed, we will now proceed with configuring Phabricator so that we can access it from the web.

### Configuring Apache

Apache is one of the supported web servers by Phabricator. Other web servers such as Nginx or lighttpd are also supported, but are out of scope of this guide. You can refer to the [official configuration guide](https://secure.phabricator.com/book/phabricator/article/configuration_guide/) for installing on those web servers.

This section will walk you through configuring Apache to point a domain to the Phabricator instance you have installed above.

1. Create a new file for storing the Apache configuration for this Phabricator instance:

        sudo nano /etc/apache2/sites-enabled/phabricator.conf

2. Set up Apache configuration to point a domain (e.g. `phabricator.example.com`) to the Phabricator instance:

    {: .file }
    /etc/apache2/sites-enabled/phabricator.conf
    :   ~~~ conf
        <VirtualHost *>
            # Change this to the domain which points to your host.
            ServerName phabricator.example.com

            DocumentRoot /mnt/phabricator/webroot

            RewriteEngine on
            RewriteRule ^/rsrc/(.*)     -                       [L,QSA]
            RewriteRule ^/favicon.ico   -                       [L,QSA]
            RewriteRule ^(.*)$          /index.php?__path__=$1  [B,L,QSA]
        </VirtualHost>
        ~~~

3. Allow Apache to read Phabricator's files located in Phabricator's web root:

    {: .file-excerpt }
    /etc/apache2/apache2.conf
    :   ~~~ conf
        <Directory "/mnt/phabricator/webroot">
            Order allow,deny
            Allow from all
        </Directory>
        ~~~

4. Restart Apache to load the new configuration settings:

        sudo service apache2 restart

If Apache restarts successfully, you should be able to navigate to the domain you have pointed to Phabricator above (e.g. `phabricator.example.com`) and see further instructions to continue with setting up Phabricator.

### Configuring MySQL

Phabricator uses MySQL as its database backend for storing objects. We will now need to install Phabricator's various databases into MySQL so that Phabricator can function.

1. Change the working directory to the location we installed Phabricator itself:

        cd /mnt/phabricator

2. Tell Phabricator which user to use for accessing the MySQL server:

        ./bin/config set mysql.user <user>

3. Provide Phabricator with the password for the user you have specified in the previous step:

        ./bin/config set mysql.pass <password>

4. Load Phabricator's databases into MySQL:

        ./bin/storage upgrade

If the above command executes successfully, all the required Phabricator databases would have been loaded into the MySQL server. You can now use Phabricator by navigating to the domain configured earlier (e.g. `phabricator.example.com`).

### Configuring Phabricator

By now, all major obstacles preventing Phabricator from loading should have been removed. When you navigate to your Phabricator instance, you should be able to see an account-creation screen. This will allow you to create an account with full administrator capabilities, allowing you to further configure Phabricator from the web interface.

After creating the administrator account, you will be greated with the default landing page for Phabricator. You will also be notified of various setup issues that you will need to resolve for certain functions of Phabricator to work properly or perform better. Each issue will provide documentation on how to solve them.

Congratulations! You now have a working instance of Phabricator! Continue reading on for documentation on getting certain complicated aspects of Phabricator working.

## Setting up repository hosting

Phabricator provides Git, SVN and Mercurial repository hosting through the Diffusion application. However, such a functionality is not available by default as there are certain additional configuration required to be done on the server-side before repository hosting can be done.

This section will walk you through setting up repository hosting on Phabricator so that your users can clone repositories on Diffusion and push changes to them over SSH and HTTPS. There are also additional help available on [Phabricator upstream](https://secure.phabricator.com/book/phabricator/article/diffusion_hosting/).

### Create system user accounts

Phabricator requires 3 different system user accounts for performing various tasks. They are described as follows:

- A user that the webserver runs as (which should already exist as `www-data` when using Apache).
- A user that the Phabricator daemons run as.
- A user for your users to connect over SSH as (for cloning of repositories over SSH).

This section will walk you through creating these users for use on Phabricator.

1. Create a user for Phabricator daemons (we will use `phab-daemon` as an example):

        sudo adduser phab-daemon

2. Create a user for SSH connections (we will use `vcs` as an example):

        sudo adduser vcs

3. Open the `/etc/sudoers` file for modification:

        sudo sudoedit /etc/sudoers

4. Allow the `www-data` and `vcs` users to `sudo` and become `phab-daemon` by modifying the `/etc/sudoers` file:

    {: .file-excerpt }
    /etc/sudoers
    :   ~~~ conf
        www-data ALL=(phab-daemon) SETENV: NOPASSWD: /usr/lib/git-core
        vcs ALL=(phab-daemon) SETENV: NOPASSWD: /usr/bin/git-upload-pack, /usr/bin/git-receive-pack
        ~~~

   You can also add the paths to `hg` and `svnserve` for the `vcs` user if your Phabricator instance uses Mercurial and Subversion respectively.

5. Open the `/etc/shadow` file for modification:

        sudo nano /etc/shadow

6. Find the line containing `vcs` and modify it such that logins can be allowed for this user:

    {: .file-excerpt }
    /etc/shadow
    :   ~~~ conf
        vcs:NP:17082:0:99999:7:::
        ~~~

   You only need to ensure that the second field is set to `NP` instead of any other value. The rest of the values can be left untouched.

7. Open the `/etc/passwd` file for modification:

        sudo nano /etc/passwd

8. Ensure that the line containing `vcs` should be set to a valid shell instead of something like `/bin/false`:

    {: .file-excerpt }
    /etc/passwd
    :   ~~~ conf
        vcs:x:1002:1002:,,,:/home/vcs:/bin/bash
        ~~~

### Configure Phabricator to use the new accounts

This section will set up Phabricator to use the new system user accounts

1. Change the working directory to the Phabricator's install path:

        cd /mnt/phabricator

2. Update Phabricator's configuration to use `phab-daemon` as the new user for the Phabricator daemons:

        ./bin/config set phd.user phab-daemon

3. Update Phabricator's configuration to use `vcs` as the user for performing SSH functionality:

        ./bin/config set diffusion.ssh-user vcs

### Configure SSH repository cloning

Repository cloning over SSH requires using the SSH port 22. While it is possible to configure Phabricator to use other ports, the clone URLs will look ugly and may even be difficult to remember if the port number used is confusing.

This section will walk you through modifying the default `sshd` process to use a different port and setting up the SSH repository cloning to use port 22.

1. Open the `/etc/ssh/sshd_config` file:

        sudo nano /etc/ssh/sshd_config

2. Modify the port that the SSH daemon listens to (e.g. to port 2222):

    {: .file-excerpt }
    /etc/ssh/sshd_config
    :   ~~~ conf
        # What ports, IPs and protocols we listen for
        Port 2222
        ~~~

3. Restart the SSH daemon:

        sudo service ssh restart

4. Ensure that you can log into your Linode machine using the new port 2222. If you are working on a Linux machine (replace USER with your own username when accessing the instance and IPADDRESS with the IP address to your Linode machine):

        ssh -p 2222 USER@IPADDRESS

5. Ensure that the old port 22 is no longer usable:

        ssh -p 22 USER@IPADDRESS

6. Create a new file in `/usr/lib` for storing Phabricator's SSH hooks file (e.g. `phabricator-ssh-hook.sh`):

        sudo nano /usr/lib/phabricator-ssh-hook.sh

7. Populate the file with the following contents (based on [upstream Phabricator template](https://secure.phabricator.com/diffusion/P/browse/master/resources/sshd/phabricator-ssh-hook.sh)):

    {: .file }
    /usr/lib/phabricator-ssh-hook.sh
    :   ~~~ shell
        #!/bin/sh

        # NOTE: Replace this with the username that you expect users to connect with.
        VCSUSER="vcs"

        # NOTE: Replace this with the path to your Phabricator directory.
        ROOT="/mnt/phabricator"

        if [ "$1" != "$VCSUSER" ];
        then
          exit 1
        fi

        exec "$ROOT/bin/ssh-auth" $@
        ~~~

8. Change the file permissions for the file we have just created:

        sudo chown root /usr/lib/phabricator-ssh-hook.sh
        sudo chmod 755 /usr/lib/phabricator-ssh-hook.sh

9. Create a new file in `/etc/ssh` for storing Phabricator's SSH configuration file (e.g. `sshd_config.phabricator`:

        sudo nano /etc/ssh/sshd_config.phabricator

10. Populate the file with the following contents (based on [upstream Phabricator template](https://secure.phabricator.com/diffusion/P/browse/master/resources/sshd/sshd_config.phabricator.example)):

    {: .file }
    /etc/ssh/sshd_config.phabricator
    :   ~~~ conf
        # NOTE: You must have OpenSSHD 6.2 or newer; support for AuthorizedKeysCommand
        # was added in this version.

        # NOTE: Edit these to the correct values for your setup.

        AuthorizedKeysCommand /usr/lib/phabricator-ssh-hook.sh
        AuthorizedKeysCommandUser vcs
        AllowUsers vcs

        # You may need to tweak these options, but mostly they just turn off everything
        # dangerous.

        Port 22
        Protocol 2
        PermitRootLogin no
        AllowAgentForwarding no
        AllowTcpForwarding no
        PrintMotd no
        PrintLastLog no
        PasswordAuthentication no
        AuthorizedKeysFile none

        PidFile /var/run/sshd-phabricator.pid

11. Once both files have been created, you can start the Phabricator's SSH daemon:

        sudo sshd -f /etc/ssh/sshd_config.phabricator

12. You can test if everything is working normally by doing:

        echo {} | ssh vcs@phabricator.example.com conduit conduit.ping

    The above would return a response such as:

        {"result":"phabricator.example.com","error_code":null,"error_info":null}

Congratulations! Your users can now use SSH to authenticate and clone repositories from Diffusion. Ensure that they have uploaded their SSH public keys into their account settings page before they begin cloning repositories.

### Configuring HTTP repository cloning

This section will walk you through setting up cloning repositories over HTTP.

{: .note}
>
> Cloning repositories over HTTP is less secure than SSH as cloning repositories over HTTP requires a VCS password, which can be stored in plaintext on the server and can be accessed by anyone.

1. Change the working directory to Phabricator's install path:

        cd /mnt/phabricator

2. Enable the configuration setting in Phabricator:

        ./bin/config set diffusion.allow-http-auth true

3. Configure a VCS password in your account settings.

4. Clone a repository hosted on Diffusion over HTTP to ensure that everything is working.
