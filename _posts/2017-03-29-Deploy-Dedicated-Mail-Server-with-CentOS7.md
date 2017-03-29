---
layout: post
title: Deploy a Dedicated E-Mail Server with CentOS 7
---
# Exim (SMTP), Dovecot (IMAP,POP3), Centos 7
This guide will help you configure an E-Mail server for your domain.  The configuration choices made here are appropriate for a smaller user base.  

This guide is a bit exhaustive.  We're going to cover purchasing a dedicated cloud server, DNS configuration, securing the server, configuration of the server, and spam protection. We optionally cover servicing mail for multiple domain names from one server.

* *Operating System*: [CentOS 7](https://www.centos.org/) .
* *Dedicated Server*:  I prefer [Linode](https://www.linode.com/?r=2bf85d3be19f1028a31c2d0e8ab1be418bf9760b) 
* *SMTP Software*: [Exim](http://www.exim.org/) 
* *IMAP/POP3*: [Dovecot](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwiNidLp2_rSAhXGYyYKHatSBhwQFggcMAA&url=https%3A%2F%2Fwww.dovecot.org%2F&usg=AFQjCNGgF9NNNYsH5m3spKRM5x_NfB3d_w&sig2=fD3TeofqOmOqxcMxVLj1yg) 
* *Account Management*: Local linux users


---
## Please Note
* example.com is the domain name referenced in the guide.  Of course, where appropriate replace this with your actual domain name
* Anywhere you see `<Server IP Address>` used in commands, please replace this with your server's actual IP address before running the command.  Any time you forget, imagine me taunting you.
* Also, I was going to use Postfix, but CentOS hasn't updated the Postfix package in years.


---
## Server Prep: Provisioning a linode dedicated server
I prefer my E-Mail server to handle only E-Mail.  [Linode](https://www.linode.com/?r=2bf85d3be19f1028a31c2d0e8ab1be418bf9760b) $5/mo Linode1024 server option is perfect for this.  

Once you've purchased the server, visit your server dashboard.  Click on Deploy an Image.   Choose CentOS 7 as your linux distribution.  Choose a [strong root password](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=0ahUKEwjTvvCk2frSAhUCKyYKHdl6CaoQFggoMAI&url=http%3A%2F%2Fwww.infoworld.com%2Farticle%2F2616157%2Fsecurity%2Fcreating-strong-passwords-is-easier-than-you-think.html&usg=AFQjCNFZO0GtsWIGXvjZtoSJvK3SjlM9ZA&sig2=P-MUmhJeWZpFTBqGTQ-ktw) .  Press the Deploy button.

Return to your server dashboard, and press the Boot button to power on your server.  Find your server's IP address under the Remote Access tab of the dashboard. 

*Make a note of this IP address.*  Throughout this guide, you will need to replace the text `<Server IP Address>` with the ip address you noted here.


---
## DNS Prep: Configure your DNS server
Other mail servers know where to send your mail by consulting your DNS zone.  Most mail servers refuse to accept email sent by your users unless your DNS zone correctly identifies your mail server, and authorizes it to send mail on your domain's behalf.

If your DNS records are not correct, you will not be able to properly send or receive mail from other servers on the internet.

It may take up to 24 hours for updates to your DNS to complete.  But usually, 30-120 minutes should suffice.  

### DNS: Properly identify your new server
Other servers will mistake you for a dirty spammer unless your DNS records are in order.  This means both forward and reverse lookups much match.

A *Forward Lookup* is when somebody asks "What is the IP Address for mail.example.com" and gets the answer 1.2.3.4.  

A *Reverse Lookup* is when somebody asks "What is the domain name for 1.2.3.4" and gets the answer mail.example.com.  

You can check these using the `host` command from your shell:

`host mail.example.com`
might display:
```
mail.example.com has address 1.2.3.4
mail.example.com has IPv6 address 1234.5678.90ab.cdef.1234.5678
```


`host 1.2.3.4` 
might display:
```
4.3.2.1.in-addr.arpa domain name pointer mail.example.com.
```

### Add DNS zone records for Forward Lookup
This is done differently depending on which name registrar you use.  Log into your registrar and find the screen for managing DNS records.
* [Name Cheap](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=12&cad=rja&uact=8&ved=0ahUKEwi44ZHU3vrSAhWBSSYKHRpJBXwQFghPMAs&url=https%3A%2F%2Fwww.namecheap.com%2Fsupport%2Fknowledgebase%2Farticle.aspx%2F319%2F78%2Fhow-can-i-setup-an-a-address-record-for-my-domain&usg=AFQjCNHJTJjINM2hTCuYYkbRTLnEdMwezQ&sig2=IeRNuyz0LsjfMQ_0B8FAOw)
* [Network Solutions](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=13&cad=rja&uact=8&ved=0ahUKEwid_aPF3vrSAhUFTSYKHTEICbYQFghPMAw&url=http%3A%2F%2Fwww.networksolutions.com%2Fsupport%2Fhow-to-manage-advanced-dns-records%2F&usg=AFQjCNGxqKezkPZVhuINHSaZu_5xn-w-nQ&sig2=oc-FNTzMdPsjHWQIdnZebQ&bvm=bv.150729734,d.eWE)
* [Go Daddy](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=0ahUKEwjF-Yvs3vrSAhWHbSYKHb9xB_cQFggiMAI&url=https%3A%2F%2Fwww.godaddy.com%2Fhelp%2Fadd-an-a-record-19238&usg=AFQjCNHofCl6HSKxmAUiIK5XlWPjt0zagg&sig2=r-n8in5e2BwdKeZIt6WaFg)


* Add an A record.
	* *Type*: A 
	* *Host*: mail
	* *Value*: <Server IP Address>
* Add an AAAA record
	* *Type*: AAAA 
	* *Host*: mail
	* *Value*: <Server IPv6 Address>
* Add an  [SPF record](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=13&cad=rja&uact=8&ved=0ahUKEwid18Kl4PrSAhVGRSYKHSZiAiEQFghWMAw&url=http%3A%2F%2Fwww.openspf.org%2F&usg=AFQjCNFhgAYms-nxqlwBs3rxT22bibAnTw&sig2=pVqUxUV6c9_nFl81wUj_nA) 
	* *Type*: TXT
	* *Host*: @
	* *Value*: v=spf1 a a:mail.example.com -all
* Add an MX record
	* *Type*: MX
	* *Host*: @
	* *Value*: mail.example.com. 10 

### Set DNS Reverse Lookup
This is different with every server provider.  Linode makes this easy.  Visit the Remote Access tab of your server dashboard.  Under the Public IPs section you will find a link for *Reverse DNS*.  Here you can specify what name should be returned when somebody looks up the IP address of your server.  Linode will not allow you to set this value until the last step has been properly configured.  It may take as long as 24 hours after setting your forward lookup records before linode will allow you to set your reverse lookup records.  If forward lookups aren't correctly configured, you will not be able to complete this step.


---
## Server Prep: Updates and Firewall
Once your  server has booted up, connect as root via SSH.  All the commands below assume you are logged in as root user.

SSH is under constant attack on the internet.  I prefer not to worry about somebody guessing or stealing my password.  I also prefer not to worry about unknown hacks against SSH. Therefore you are going to block SSH for everybody except from your own whitelisted IP address.

If you don't know your public IP address, you can ask google `what is my ip`

### Activate the firewall
``` sh
# Activate firewall
systemctl start firewalld
systemctl enable firewalld
```

### Whitelist my trusted IP
You can repeat this for as many IP addresses as you wish to trust.  Replace 1.2.3.4 with your trusted address.
```
# Trust my ip 
firewall-cmd --zone=trusted --add-source=1.2.3.4
firewall-cmd --zone=trusted --add-source=1.2.3.4 --permanent
```

### Block SSH trafic from the public
Your trusted IP address can still access SSH
```
# Disable ssh to the world (my trusted ip can still ssh)
firewall-cmd --zone=public --remove-service=ssh --permanent
firewall-cmd --zone=public --remove-service=ssh
```

### Open mail ports in firewall
```
# Open the mail ports now
firewall-cmd --zone=public --add-service=smtp
firewall-cmd --zone=public --add-service=smtps
firewall-cmd --zone=public --add-service=imap
firewall-cmd --zone=public --add-service=imaps
firewall-cmd --zone=public --add-service=pop3
firewall-cmd --zone=public --add-service=pop3s

# Open the mail ports after reboots
firewall-cmd --zone=public --add-service=smtp --permanent
firewall-cmd --zone=public --add-service=smtps --permanent
firewall-cmd --zone=public --add-service=imap --permanent
firewall-cmd --zone=public --add-service=imaps --permanent
firewall-cmd --zone=public --add-service=pop3 --permanent
firewall-cmd --zone=public --add-service=pop3s --permanent

```


---
## If your Public IP changes and you are locked out of SSH!
This is no problem.  Log into your linode dashboard at [linode.com](https://www.linode.com/?r=2bf85d3be19f1028a31c2d0e8ab1be418bf9760b).  Under the Remote Access tab, you find an option called *Launch Lish Console via Browser*.  This will give you an SSH shell to your server in your web browser.  Log in as root from here., and run the above trust firewall commands to permit your new public ip through your firewall. 


---
## Server Prep: Hostname and Updates
Your server must know it's own name.   Replace the example domain and ip with your domain and ip.
``` bash
# Set the hostname
hostnamectl set-hostname mail.example.com

# Set the domainname
domainname mail.example.com

# Update local hosts file
cat "1.2.3.4 mail.example.com" >> /etc/hosts
```

Update the server
```
# updates
yum -y install epel-release
yum -y update
yum clean all

```


---
## Bonus Points: Track configuration changes with git
If you're familiar with git, use it to track configuration changes.
I email myself the results of `git status`  on a daily basis.
``` sh
yum -y install git
git config --global user.name "Mitch"
git config --global user.email "mitch@example.com"
cd /etc
git init
git add .
git commit -m "Initial /etc configuration"
```

What files have changed?
`git status`

What changes were made in those files?
`git diff`

Commit changes every time I update the server configuration
``` sh
git add .
git commit -m "Update the server"
```

View of history of changes
`git log`



---
## Generate a free SSL Certificate
I like my E-Mail to travel the internet via SSL encrypted channels whenever possible.  For this, you need a valid SSL certificate.  The [Let's Encrypt Project](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwiJ-fGn5_rSAhWEQyYKHVipA80QFggaMAA&url=https%3A%2F%2Fletsencrypt.org%2F&usg=AFQjCNHLSK55fO3YLnyT_sYlhlFMyyILjg&sig2=EcwJsM2n80UP-ENfpiLAKQ&bvm=bv.150729734,d.eWE) was kind enough to make this very easy.  

The certificate generator is going to verify your domain name before issuing a certificate.    This isn't going to work if your DNS isn't properly configured.

``` sh
# install letsencrypt
yum -y install certbot

# Allow letsencrypt through the firewall
systemctl stop firewalld

# Request an SSL certificate
certbot certonly \
        -d mail.example.com \  
        --standalone \
        -n \
        -m mitch@example.com \
        --agree-tos

# Restore the firewall
systemctl start firewalld

# The exim and dovecot users must be able to
# read the ssl certificates.  Loosen permissions.
chmod a+rx /etc/letsencrypt/live 
chmod a+rx /etc/letsencrypt/archive

```



---
## Exim: Install and Configure
### Install the SMTP server
`yum -y install exim`

### /etc/exim/exim.conf
Make several changes to this large configuration file.  For each change, locate the appropriate lines in the file and update accordingly.

Set the full mail server domain name.  This needs to match the DNS.
`primary_hostname = mail.example.com`

Specify which top level domains we receive mail for
`domainlist local_domains = @ : localhost : localhost.localdomain : example.com`

Set the maximum size for E-Mail attachments
The Default is 50M.  I've doubled it here
```
# Add this line after the domainlist line near the top of the file
message_size_limit=100M
```

Specify the SSL certificate
```
tls_certificate = /etc/letsencrypt/live/mail.example.com/fullchain.pem
tls_privatekey  = /etc/letsencrypt/live/mail.example.com/privkey.pem
```

Uncomment this line, so we can use linux account passwords to authenticate mail users
`auth_advertise_hosts = ${if eq {$tls_cipher}{}{}{*}}`

Also comment out this line, to enable the above line:
`# auth_advertise_hosts = `

Ask the SMTP server to store e-mail messages for the mail users
in their account folders, by using this local_delivery section
```
local_delivery:
  driver = appendfile
  file = $home/Maildir
  maildir_format 
  maildir_use_size_file
  delivery_date_add
  envelope_to_add
  return_path_add

```

Exim is going to let Dovecot do the work of authenticating users for login attempts.
Find the `authenticators` section, and add these two sections
```
# Append to authenticators section

dovecot_login:
  driver = dovecot
  public_name = LOGIN
  server_socket = /var/run/dovecot/auth-client
  server_set_id = $auth1

dovecot_plain:
  driver = dovecot
  public_name = PLAIN
  server_socket = /var/run/dovecot/auth-client
  server_set_id = $auth1
```


### Notify linux we're using exim instead of sendmail
`alternatives --set mta /usr/sbin/sendmail.exim`

### Test the exim config file
If a typo has invalidated the exim configuration file, this will help you find the problem
`exim -C /etc/exim/exim.conf -bV`

### Enable Exim as as service
Start Exim, enable starting Exim on boot
```
systemctl start exim
systemctl enable exim
```


---
## Install and Configure Dovecot
`yum -y install dovecot`

### /etc/dovecot/conf.d/10-ssl.conf
Configure SSL cert for dovecot
```
ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.example.com/privkey.pem
```

### /etc/dovecot/conf.d/10-auth.conf
All connections are secured by SSL.  Allow plaintext logins over ssl 
```
disable_plaintext_auth=no
auth_mechanisms = plain login
```

### /etc/dovecot/conf.d/10-mail.conf
Set  user mailbox location.
```
mail_location = maildir:~/Maildir
```

### /etc/dovecot/conf.d/10-master.conf
Allow exim to authenticate via dovecot.
The unix_listener auth-client block must be added inside the service auth block
```
service auth {
# ...
# ...
    unix_listener auth-client {
        mode = 0660
        user = exim
    }
# ...
# ...
}
```

### Enable Dovecot as a service
```
systemctl enable dovecot
systemctl start dovecot
```


---
## Adding and managing users
Your E-Mail server will service a mailbox for every local user on the system. Therefore, to add e-mail accounts, just create a system user and set the system user password.  To remove an e-mail account, delete the system user.

When creating the user, I prefer to set their group as exim, and their shell as /usr/sbin/nologin.  This ensures shared file access with the mail server, and stops the user from actually logging into the server somehow.

### Creating mail accounts
``` sh
# for me@example.com
adduser -g exim -s /usr/sbin/nologin me
passwd me

# for mitch@mitchjacksontech.com
adduser -g exim -s /usr/sbin/nologin mitch
passwd mitch
```

### Deleting mail accounts
``` sh
userdel -r me
userdel -r mitch
```

### System Aliases /etc/aliases
The aliases applied in /etc/aliases will also apply to your e-mail system.  As the sysadmin, if I want to receive all the system mail generated for the root user, update that alias here. 

It would be considered good form to keep up with the postmaster address.  It's going to be 99.9999999% spam.  But somebody might email you about a delivery problem that is legitimate.  You can alias that mailbox to your actual mail account here.

If you want mitch@example.com to receive mail for  mcp@example.com, help@example.com and rickroll@example.com, add entries to /etc/aliases and restart the mail server.


---
## Troubleshooting
### Check the exim config file
`exim -C /etc/exim/exim.conf -bV`

### DNS Problems
If DNS or SPF isn't correct, there will be problems sending and/or receiving mail.  You can use the `host` tool to verify your settings.  Many websites exist to help you check your DNS records are correct.  You may find [mxtoolbox](https://mxtoolbox.com/) or [network-tools](http://network-tools.com/) helpful.

### Testing mail routing
Ask exim where mail goes if somebody sends to an e-mail address.  
`exim4 -bt mitch@example.com`

### Check the logs
There's a ton of useful fun in the log files.  I like to watch them scroll bywhen the server is healthy. They are informative when it is not.  Running the following set of commands will show you log entries as they happen in your terminal. 
``` sh
# attack of the logs
#

# General server logs
journalctl -f &
tail -f /var/log/secure &
tail -f /var/log/messages &

# mail logs
tail -f /var/log/maillog &
tail -f /var/log/exim/main.log &
tail -f /var/log/exim/panic.log &
tail -f /var/log/exim/reject.log &
```

### selinux
[selinux](https://selinuxproject.org) is probably interfering with your mail server, unless you know how to use it.  If you want to understand selinux, find somebody who doesn't hate it to explain.  Otherwise, just disable it by editing /etc/selinux/config:
`SELINUX=disabled`


---
## Handling mail for multiple domain names
You can accomplish this goal with very little additional configuration.  There are more complicated configuration paths for this involving databases and boatloads of extra work. Here we're keeping it simple.

Every user of your mail server continues to be a local system user.  If you direct exim to handle multiple top level domains using local unix accounts, you must specify which local user account is for which e-mail address.

You can support a large number of users and domain names using this method, but the usernames may get messy unless you have an organized approach from the start.

### /etc/exim/aliases.virtual
Create this file, and populate it with a map of local users and e-mail addresses.  This file can also be used to map multiple e-mail addresses to one user account.  For example:
``` sh
# /etc/exim/aliases.virtual
#
# Line Syntax:
# <E-Mail Address> : <local user>

# example.com
mitch@example.com : mitch
admin@example.com : mitch
paul@example.com  : paul
help@example.com  : paul

# testdomain.net
mitch@testdomain.net : mitch_tdn
andy@testdomain.net  : andy
paul@testdomain.net  : paul
```

Since we have two people named Mitch using our server, one username is mitch.  The other gets the username mitch_tdn.  Paul chose to only have one account, paul@example.net, and have the email alias paul@testdomain.net.

### /etc/exim/exim.conf
Each domain we will receive mail for must be added to this line of the config file:
`domainlist local_domains = @ : localhost : localhost.localdomain : example.com : testdomain.net : bagofcrap.com `

In the routers section, tell exim where to find the account map you made
Place this section inside the routers section, directly before the localuser router
```
# consult the alias map
domains_virtual:
  driver = redirect
  data = ${lookup{$local_part@$domain}lsearch{/etc/exim/aliases.virtual}}

```

Exim has good intentions here, but we must intervene.  Find the section below for authenticated user relay control, and modify it to match.  Otherwise, when andy@testdomain.net sends an email, exim will irritatingly rewrite the sender address as andy@example.com before the message is delivered.
*/etc/exim/exim.conf*
``` sh
# Find the relay control for authenticated users
# and add 'sender_retain' where shown:
accept  authenticated = *
          control       = submission/sender_retain
          control       = dkim_disable_verify
```


---
## Spam control
A large volume of spam can be discarded through simple sanity checks, and blacklist checks.  I don't cover heuristic spam detection such as [spam assassin](http://spamassassin.apache.org/) .   As you grow to more users with more spam, you may need more desperate measures.  These controls have worked very well for me.

All these changes will be made to the ACL lists inside the specified section exim.conf 

### acl_check_mail
```
  # If the sender name contains an ip address
  drop message        = Helo name contains a ip address (HELO was $sender_helo_name) and not is valid
       condition      = ${if match{$sender_helo_name}{\N((\d{1,3}[.-]\d{1,3}[.-]\d{1,3}[.-]\d{1,3})|([0-9a-f]{8})|([0-9A-F]$
       condition      = ${if match {${lookup dnsdb{>: defer_never,ptr=$sender_host_address}}}{$sender_helo_name}{no}{yes}}
       delay          = 45s


  # HELO is neither FQDN nor address literal
  drop
    # Required because "[IPv6:<address>]" will have no .s
    condition   = ${if match{$sender_helo_name}{\N^\[\N}{no}{yes}}
    condition   = ${if match{$sender_helo_name}{\N\.\N}{no}{yes}}
    message     = Access denied - Invalid HELO name (See RFC2821 4.1.1.1)

  # HELO is forging my hostname
  drop message   = "REJECTED - Bad HELO - Host impersonating [$sender_helo_name]"
       condition = ${if match{$sender_helo_name}{$primary_hostname}}

  # HELO is faked interface address
  drop condition = ${if eq{[$interface_address]}{$sender_helo_name}}
     message   = $interface_address is _my_ address


```


### acl_check_rcpt
```
# check for sender blacklist
deny message = Access denied - $sender_host_address\
               listed by $dnslist_domain\n$dnslist_text
     dnslists = dnsbl.sorbs.net:sbl.spamhaus.org:bl.spamcop.net:cbl.abuseat.org

# Drop invalid bounces
drop  message = Legitimate bounces are never sent to more than one recipient.
      senders = : postmaster@*
      condition = ${if >{$recipients_count}{0}{true}{false}}

```


---
## It's an E-Mail Party
You should now have your own E-Mail server.  [drop me a line](mailto:mitch@mitchjacksontech.com) and let me know how it's working out for you.

#linux
