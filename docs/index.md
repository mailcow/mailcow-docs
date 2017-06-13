<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
![mailcow](https://www.debinux.de/256.png)

- [mailcow](#mailcow)
- [Get Support](#get-support)
- [Introduction](#introduction)
- [System Requirements](#system-requirements)
- [Before You Begin (Prerequisites)](#before-you-begin-prerequisites)
- [Installation](#installation)
- [Upgrade](#upgrade)
- [Uninstall](#uninstall)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

mailcow
=====

mailcow is a mail server suite based on Dovecot, Postfix and other open source software, that provides a modern web UI for user/server administration.

mailcow supports **Debian 8 (Jessie), Ubuntu LTS 14.04 (Trusty Tahr) and Ubuntu LTS 16.04 (Xenial Xerus)**

[Everybody loves screenshots (v0.14)](http://imgur.com/a/lWX2V)

## Get support

### Commercial support

For commercial support contact [info@servercow.de](mailto:info@servercow.de).

### Community support

- IRC @ [Freenode, #mailcow](irc://irc.freenode.org:6667/mailcow)
- Forum @ [forum.mailcow.email](forum.mailcow.email)
- GitHub @ [mailcow/mailcow-dockerized](https://github.com/mailcow/mailcow)

# Introduction

* Multi-SAN self-signed SSL certificate for all installed and supporting services
    * Let's Encrypt optional
* Webserver installation
    * Apache or Nginx (+PHP5-FPM)
* SQL database backend, remote database support
    * MySQL or MariaDB
* **mailcow web UI**
    * Add domains, mailboxes, aliases, set limits, enforce TLS outgoing and incoming, monitor mail statistics, change mail server settings, create/delete DKIM records and more...
* Postscreen activated and configured
* STARTTLS and SMTPS support
* The default restrictions used are a good compromise between blocking spam and avoiding false-positives
* Incoming and outgoing spam and virus protection with FuGlu as pre-queue content filter; [Heinlein Support](https://www.heinlein-support.de/) spamassassin rules included; Advanced ClamAV malware filters
* Sieve/ManageSieve (default filter: move spam to "Junk" folder, move tagged mail to folder "tag")
* Public folder support via control center
* Per-user ACL
* Shared Namespace
* Quotas
* Auto-configuration for ActiveSync + Thunderbird (and its derivates)

Comes with...
* Roundcube
    * ManageSieve support (w/ vacation)
    * Attachment reminder (multiple locales)
    * Zip-download marked messages
or
* SOGo
    * Full groupware with ActiveSync and Card-/CalDAV support

# System Requirements
Please check, if your system meets the following system resources:

| Resource                | mailcow (Roundcube) | mailcow (SOGo) |
| ----------------------- | ------------------- | -------------- |
| CPU                     | 1 GHz               | 1 GHz          |
| RAM                     | 800 MiB             | 1 GiB          |
| Disk                    | 5 GiB               | 5 GiB          |
| System Type             | x86 or x86_64       | x86_64         |

# Before You Begin: Prerequisites
- **Please remove any web and mail services** running on your server. I recommend using a clean Debian minimal installation.
Remember to purge Debian's default MTA Exim4:
```
apt-get purge exim4*
```

- If there is a firewall, unblock the following ports for incoming connections:

| Service               | Protocol | Port   |
| -------------------   |:--------:|:-------|
| Postfix Submission    | TCP      | 587    |
| Postfix SMTPS         | TCP      | 465    |
| Postfix SMTP          | TCP      | 25     |
| Dovecot IMAP          | TCP      | 143    |
| Dovecot IMAPS         | TCP      | 993    |
| Dovecot POP3          | TCP      | 110    |
| Dovecot POP3S         | TCP      | 995    |
| Dovecot ManageSieve   | TCP      | 4190   |
| HTTP(S)               | TCP      | 80/443 |

- Setup DNS records:

Obviously you will need an A and/or AAAA record `sys_hostname.sys_domain` pointing to your IP address and a valid MX record.
Let's Encrypt does not assign certificates when it cannot determine a valid IPv4 address.

| Name                       | Type   | Value                        | Priority   |
| ---------------------------|:------:|:----------------------------:|:-----------|
| `sys_hostname.sys_domain`  | A/AAAA | IPv4/6                       | any        |
| `sys_domain`               | MX     | `sys_hostname.sys_domain`    | 25         |

Optional: Auto-configuration services for Thunderbird (and derivates) + ActiveSync.
You do not need to setup `autodiscover` when not using SOGo with ActiveSync.

| Name                       | Type   | Value                        | Priority   |
| ---------------------------|:------:|:----------------------------:|:-----------|
| autoconfig.`sys_domain`    | A/AAAA | IPv4/6                       | any        |
| autodiscover.`sys_domain`  | A/AAAA | IPv4/6                       | any        |

**Hint:** ActiveSync auto-discovery is setup to configure desktop clients with IMAP!

Further DNS records for SPF and DKIM are recommended. These entries will raise trust in your mailserver, reduce abuse of your domain name and increase authenticity.

Find more details about mailcow DNS entries and SPF/DKIM related configuration in our wiki article on [DNS Records](https://github.com/andryyy/mailcow/wiki/DNS-records).

- Next it is important that you **do not use Google DNS** or another public DNS which is known to be blocked by DNS-based Blackhole List (DNSBL) providers.

I recommend PowerDNS Recursor as a local recursor with DNSSEC capabilities. See https://repo.powerdns.com/.
Though any non-blocked or rate limited DNS server your ISP gave you *should* be fine.

# Installation
**Please run all commands as root**

**Download a stable release**

Download mailcow to whichever directory (using ~/build here).
Replace "v0.x" with the tag of the latest release: https://github.com/andryyy/mailcow/releases/latest
```
mkdir ~/build ; cd ~/build
wget -O - https://github.com/andryyy/mailcow/archive/v0.x.tar.gz | tar xfz -
cd mailcow-*
```

**Now edit the file "configuration" to fit your needs!**
```
nano mailcow.config
```

* **sys_hostname** - Hostname without domain
* **sys_domain** - Domain name. "$sys_hostname.$sys_domain" equals to FQDN.

:exclamation: Please make sure your FQDN resolves correctly!

* **sys_timezone** - The timezone must be defined in a valid format (Europe/Berlin, America/New_York etc.)
* **use_lets_encrypt** - Tries to obtain a certificate from Let's Encrypt CA. If it fails, it can be retried by calling `./install.sh -s`. Installs a cronjob to renew the certificate but keeps the same key.
* **httpd_platform** - Select wether to use Nginx ("nginx") or Apache2 ("apache2"). Nginx is default.
* **mailing_platform** - Can be "sogo" or "roundcube"
* **my_dbhost** - ADVANCED: Leave as-is ("localhost") for a local database installation. Anything but "localhost" or "127.0.0.1" is recognized as a remote installation.
* **my_usemariadb** - Use MariaDB instead of MySQL. Only valid for local databases. Installer stops when MariaDB is detected, but MySQL selected - and vice versa.
* **my_mailcowdb, my_mailcowuser, my_mailcowpass** - SQL database name, username and password for use with Postfix. **You can use the default values.**
* **my_rcdb, my_rcuser, my_rcpass** - SQL database name, username and password for Roundcube. **You can use the default values.**
* **my_rootpw** - SQL root password is generated automatically by default. You can define a complex password here if you want to. *Set to your current root password to use an existing SQL instance*.
* **mailcow_admin_user and mailcow_admin_pass** - mailcow administrator. Password policy: minimum length 8 chars, must contain uppercase and lowercase letters and at least 2 digits. **You can use the default values**.
* **inst_debug** - Sets Bash mode -x
* **inst_confirm_proceed** - Skip "Press any key to continue" dialogs by setting this to "no"

**Empty configuration values are invalid!**

You are ready to start the script:
```
./install.sh
```
Just be patient and confirm every step by pressing [ENTER] or [CTRL-C] to interrupt the installation.
If you run into problems, try to locate the error with "inst_debug" enabled in your configuration.
Please contact me when you need help or found a bug.

More debugging is about to come. Though everything should work as intended.

After the installation, visit your dashboard @ **https://hostname.example.com**, use the logged credentials in `./installer.log`

Remember to create an alias or a mailbox for `postmaster`.

Again, please check you setup all DNS records accordingly.

## Web UI configuration variables

Some settings can be changed by overwriting defaults of `/var/www/mail/inc/vars.inc.php` in `/var/www/mail/inc/vars.local.inc.php`.

Changes to `/var/www/mail/inc/vars.local.inc.php` will not be overwritten when upgrading mailcow.

##

# Upgrade
**Please run all commands as root**

The mailcow configuration file (mailcow.config) will not be read, so there is no need to adjust it in any way before upgrading.

To start the upgrade, run the following command:
```
./install.sh -u
```

**Please double check the detected FQDN!**

When autodetection of your hostname and/or domain name fails, use the `-H` parameter to overwrite the hostname and/or `-D` to overwrite the domain name:
```
# FQDN: mx.example.org
./install -u -H mx -D example.org
```

# Uninstall
Please remove the components you do not need manually. mailcow installs components that may be used by other software on your system.
mailcow is an installer that installs and configures software, so there is no routine to remove itself.

**A list of by apt-get installed components**
```
# System tools
dnsutils sudo zip bzip2 unzip unrar-free curl openssl file bsd-mailx

# Core components
# ${OPENJDK} is either "openjdk-7" or "openjdk-9"
rrdtool mailgraph fcgiwrap spawn-fcgi python-setuptools libmail-spf-perl libmail-dkim-perl mailutils pyzor razor postfix postfix-mysql postfix-pcre postgrey pflogsumm spamassassin spamc opendkim opendkim-tools clamav-daemon python-magic liblockfile-simple-perl libdbi-perl libmime-base64-urlsafe-perl libtest-tempdir-perl liblogger-syslog-perl ${OPENJDK}-jre-headless libcurl4-openssl-dev libexpat1-dev solr-jetty

# PHP components
# ${PHP} is either PHP or PHP5
php-auth-sasl php-http-request php-mail php-mail-mime php-mail-mimedecode php-net-dime php-net-smtp php-net-socket php-net-url php-pear php-soap ${PHP} ${PHP}-cli ${PHP}-common ${PHP}-curl ${PHP}-gd ${PHP}-imap ${PHP}-intl ${PHP}-xsl libawl-php ${PHP}-mcrypt ${PHP}-mysql ${PHP}-xmlrpc

# Database components
mariadb-client mariadb-server
# or...
mysql-client mysql-server

# Webserver components
# ${PHP} is either PHP or PHP5
apache2 apache2-utils libapache2-mod-${PHP}
# or...
nginx-extras ${PHP}-fpm

# Dovecot components
dovecot-common dovecot-core dovecot-imapd dovecot-lmtpd dovecot-managesieved dovecot-sieve dovecot-mysql dovecot-pop3d dovecot-solr

# SOGo
# ALSO: rm /etc/apt/sources.list.d/sogo.list
sogo sogo-activesync libwbxml2-0 memcached

```

**System modifications**
```
# Cronjobs
rm /etc/cron.daily/mc_clean_spam_aliases /etc/cron.daily/mailcow-clean-spam-aliases /etc/cron.daily/dovemaint /etc/cron.d/solrmaint /etc/cron.daily/spamlearn /etc/cron.daily/spamassassin_heinlein /etc/cron.weekly/le-renew

# Sudo
rm /etc/sudoers.d/mailcow

# Executables
rm /usr/local/sbin/mailcow-reset-admin /usr/local/sbin/mailcow-dkim-tool /usr/local/sbin/mailcow-set-message-limit /usr/local/sbin/mailcow-renew-pflogsumm /usr/local/sbin/mc_pflog_renew /usr/local/sbin/mc_msg_size /usr/local/sbin/mc_dkim_ctrl /usr/local/sbin/mc_resetadmin
```

**Manually installed components and miscellaneous**
```
# Databases
# Besides aboves packages, you may want to drop the mailcow and, if installed, Roundcube database. SOGo uses the mailcow database.
DROP DATABASE $mailcowdb;
DROP DATABASE $roundcubedb;

# Let's Encrypt
rm -r /opt/letsencrypt-sh/

# FuGlu
systemctl disable fuglu
rm -rf /usr/local/lib/python2.7/dist-packages/fuglu*
update-rc.d -f fuglu remove
userdel fuglu
# FuGlu dependencies
python-sqlalchemy python-beautifulsoup python-mysqldb
```
