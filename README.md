# Getting Kazoo up and Running on AWS EC2 servers
*(A guide by  the Open Telecom Foundation)*

This guide is dedicated to Installing Kazoo on the AWS EC2 servers. It will be broken into 3 parts, Getting an EC2 instance using the Centos AMI up and running, ssh into EC2 instance on the linux command line, installing and configuring Kazoo on the EC2 instance.


## Getting an EC2 Instance on AWS Using AWS CLI

Once you have a AWS account set up, install AWS CLI on your machine, this guide assumes you have a private/public keyset ready to use 
(HINT : ```ssh-keygen```).

```
brew install awscli
```
After install, run

```
aws configure
```
Enter your credentials as prompted

Upload your public key to the AWS console

```
aws ec2 import-key-pair --key-name "otf_kazoo_key" --public-key-material file://~/.ssh/id_rsa.pub`
```

Output:

```
{
  "KeyName": "my-key",
  "KeyFingerprint": "1f:51:ae:28:bf:89:e9:d8:1f:25:5d:37:2d:7d:b8:ca"
}
```

## Create Security Group
```
aws ec2 create-security-group --group-name otf_kazoo_sg1 --description "OTF_kazoo-Security_group
```

**Output**:

```
{
    "GroupId": "sg-0c880c0a90352eca7"
}
```

***NOTE : otf_kazoo_sg1 : This is the name of the security group that will be used throughout***

## Create Instance on AWS

```
# change count to the number of instances you want, but one will suffice for deploying kazoo

aws ec2 run-instances --image-id ami-e1496384 --count 1 --instance-type t2.large --key-name otf_kazoo_key --security-groups otf_kazoo_sg1
```

### Add Inbound Rules to Instance

**for tcp add all the following ports**

```
aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol tcp --port 22 --cidr 0.0.0.0/0`

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol tcp --port 8000 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol tcp --port 8443 --cidr 0.0.0.0/0`

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol tcp --port 1111 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol tcp --port 2222 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol tcp --port 443 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol tcp --port 5060 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol tcp --port 7000 --cidr 0.0.0.0/0
```

**for udp add the following ports**

```
aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol udp --port 7000 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol udp --port 10000-40000 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol udp --port 5060 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol icmp --port  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-name otf_kazoo_sg1 --protocol icmp --port all --cidr 0.0.0.0/0
```


## SSH into instance

**Get public IP of instance**
``` 
aws ec2 describe-instances --instance-ids instance-id --query 'Reservations[].Instances[].PublicDnsName'
```

 # Replace instance id with your instane’s ID from your aws console

*Using Results from the terminal which should look like this*

```
[
    "ec2-3-17-74-39.us-east-2.compute.amazonaws.com"
]
```
ssh to your instance
```
ssh -i pathTo-myPrivateKEy.pem root@ec2-3-17-74-39.us-east-2.compute.amazonaws.com
```
# Installing Kazoo on the EC2 Server

## Pre-Install

**FQDN Check**

Make sure both `hostname` and `hostname -f` both return the fully qualified domain name, otherwise the procedure will fail.


**Check status**

```
sestatus
```


**If not disabled, disable Selinux and reboot**

Disable Selinux

```
sed -i 's/\(^SELINUX=\).*/\SELINUX=disabled/' /etc/selinux/config
```

```
reboot
```

**Disable Firewall**

```
systemctl disable firewalld
systemctl disable iptables
systemctl stop firewalld
systemctl stop iptables
```

**NOTE: Make sure to reinstall the Firewall at the end.**

Set Timezone

```
yum install ntp
```
```
systemctl enable ntpd
systemctl start ntpd
```

```
timedatectl set-timezone UTC
```
**Pre-requirements**

```
yum -y update
yum -y install net-tools wget gdb yum-utils bash-completion epel-release
```

## Kazoo, Kamailio and FreeSWITCH Repositories

**Install the necessary RPM repositories for the latest stable release for Kazoo**

```
cd /usr/src
wget --no-check-certificate \
```

*copy link to console*

https://packages.2600hz.com/centos/7/stable/2600hz-release/4.2/2600hz-release-4.2-0.el7.centos.noarch.rpm

```
rpm -Uvh 2600hz-release-4.2-0.el7.centos.noarch.rpm

yum-config-manager --disable 2600hz-experimental
yum-config-manager --disable 2600hz-staging
yum-config-manager --enable 2600hz-stable
```

Now run the following step by step, should take less than 15 minutes to install and update

**Install**
```
yum -y install kazoo-bigcouch kazoo-haproxy kazoo-rabbitmq kazoo-freeswitch kazoo-kamailio kazoo-applications kazoo-application-* monster-ui* httpd
```


**Enable**
```
systemctl enable kazoo-bigcouch kazoo-haproxy kazoo-rabbitmq kazoo-freeswitch kazoo-kamailio kazoo-applications kazoo-ecallmgr httpd
```

**Restart**
```
systemctl restart kazoo-bigcouch kazoo-haproxy kazoo-rabbitmq kazoo-freeswitch kazoo-kamailio kazoo-applications kazoo-ecallmgr httpd
```
```
# haproxy has no Server at this point, seeing the following message is ok
"proxy bigcouch-mgr has no server available!"


/usr/sbin/chkconfig kamailio off
```

**To ensure the database creation was completed**
```
curl localhost:15984/_all_dbs | python -mjson.tool | wc -l
```
*The Lower left number should be 26*

## Finally Configure Kazoo
```
sup kazoo_media_maintenance import_prompts /opt/kazoo/sounds/en/us/


# Create master account. Account name, realm and password can be changed afterwards via Monster UI.

sup crossbar_maintenance create_account master master.local superadmin somepassword
```

```
# use this information to log into monster-ui
master - account name
master.local - domain
superadmin - user
somepassword - password
```

```
serverIP=$(ifconfig | sed -En 's/127.0.0.*//;s/.*inet (addr:)?(([0-9]*\.){3}[0-9]*).*/\2/p')

serverFQDN=$(hostname)


sed -i "s/127\.0\.0\.1/$serverIP/g" /etc/kazoo/kamailio/local.cfg
sed -i "s/kamailio\.2600hz\.com/$serverFQDN/g" /etc/kazoo/kamailio/local.cfg
sed -i "s/localhost/$serverIP/" /var/www/html/monster-ui/js/config.js


systemctl restart kazoo-kamailio
```
Check ecallmgr node is connected to freeswitch
```
sup -n ecallmgr ecallmgr_maintenance add_fs_node freeswitch@$serverFQDN

# seeing the following error is expected and should not alarm you
{error,node_exists} 
```

**The following command is run twice because it doesn't seem to always stick the first time.**
Add Kamailio to ACL so that Freeswitch allows the traffic
```
sup -n ecallmgr ecallmgr_maintenance allow_sbc kamailio1 $serverIP
sup -n ecallmgr ecallmgr_maintenance allow_sbc kamailio1 $serverIP
```


Replace ~~serverIP:8000/v2~~ with **x.x.x.x:8000/v2**   

**x.x.x.x** being your pubic IP from AWS EC2 instance.

***This is because, while creating the $serverIP, AWS uses the internal IP and not the public one, this causes problem while loading the apps from monster-ui.***


```
sup crossbar_maintenance init_apps /var/www/html/monster-ui/apps http://$serverIP:8000/v2


# Pay extra attention while running this command, it is crucial that the
**httpd/conf.d/monster-ui.conf file get the right $serverFQDN.**

echo "<VirtualHost *:80>
DocumentRoot \"/var/www/html/monster-ui\"
ServerName $serverFQDN
</VirtualHost>
" > /etc/httpd/conf.d/monster-ui.conf
```

**check if http got the right IP**

```
cat /etc/httpd/conf.d/monster-ui.conf`

# You should be able to see your internal ip address next to ServerName

echo "<VirtualHost *:80>
DocumentRoot \"/var/www/html/monster-ui\"
ServerName  **YOUR-INTERNAL AWS IP SHOULD BE HERE!** 
</VirtualHost>
" > /etc/httpd/conf.d/monster-ui.conf


systemctl reload httpd
```

## Kazoo check

```
reboot
```
```
# After reboot, run the following test to make sure all configurations stuck. Almost 90% of the errors happen here!
```

Check that Freeswitch is connected to ecallmgr.

```
fs_cli -x 'erlang status'
```
```
**output**

Running mod_kazoo v1.4.0-1
Listening for new Erlang connections on 0.0.0.0:8031 with cookie change_me
Registered as Erlang node freeswitch@ip-172-31-9-68.us-east-2.compute.internal, visible as freeswitch
Connected to:
  ecallmgr@ip-172-31-9-68.us-east-2.compute.internal (172.31.9.68:8031) up 0 years, 1 days, 3 hours, 59 minutes, 19 seconds
```

*This might take a few minutes after reboot*

Check that Kamailio IP is in ACL.

```
sup -n ecallmgr ecallmgr_maintenance acl_summary
```

**output**

| Name                         | CIDR            | List   | Type | Authorizing Type | ID 
|-|-|-|-|-|-|
| kamailio1                    | 172.31.9.68/32  | authoritative | allow | system_config
| kamailio@kamailio.2600hz.com | 0.0.0.0/32      | authoritative | allow | system_config | 

****172.31.9.68/32 should be your internal AWS IP***

```
** edit the $serverIP and paste it manually from your AWS EC2 instance. ONLY do this if  kamailio is not listed!!**


sup -n ecallmgr ecallmgr_maintenance allow_sbc kamailio1 $serverIP
sup -n ecallmgr ecallmgr_maintenance allow_sbc kamailio1 $serverIP
```

```
Check that Freeswitch is in configuration

sup -n ecallmgr ecallmgr_maintenance get_fs_nodes
```
```
**Output**

freeswitch@ip-172-31-9-68.us-east-2.compute.internal
```

**Check Erlang nodes**

```
epmd -names


# Should result in:
epmd: up and running on port 4369 with data:
name kazoo-rabbitmq at port 25672
name freeswitch at port 8031
name kazoo_apps at port 11502
name ecallmgr at port 11501
name bigcouch at port 11500
```


```
# Check overall system status
# You should see 3 nodes listed as shown at Check status.  Kazoo Apps, Kamailio, and Ecallmgr.
# The Freeswitch media server should be listed under Ecallmgr node.
# Freeswitch IP on port 11000 with (AP) status should be listed beside Dispatcher 1 under Kamailio node.`

kazoo-applications status
```
[Check status](https://www.powerpbx.org/sites/default/files/kazoo-applications-status.txt " Check kazoo status")


## Post Install

**Firewall**

```
systemctl enable firewalld
systemctl restart firewalld
```

```
firewall-cmd --permanent --zone=public --add-service={http,https}
firewall-cmd --permanent --zone=public --add-port={8000,8443}/tcp
firewall-cmd --permanent --zone=public --add-port={5060,7000}/tcp
firewall-cmd --permanent --zone=public --add-port={5060,7000}/udp
firewall-cmd --permanent --zone=public --add-port=16384-32768/udp


#Administrator access.  Replace x.x.x.x with the public IP address of your admin computer.
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="x.x.x.x" accept'

firewall-cmd --reload
```

### SUP
**This is the Kazoo Supervisor, allows accessing Erlang from the Command line**

```
# View top level list of commands using bash completion

sup [TAB][TAB]

# To view next level down with autocomplete (using ecallmgr_maintenance as an example)

sup ecallmgr_m[TAB][TAB][TAB]

# Create a file listing all sup commands

mkdir /usr/doc
/opt/kazoo/lib/sup*/priv/build-autocomplete.escript \
/etc/bash_completion.d/sup.bash /opt/kazoo > /usr/doc/sup_commands

# View the file
cat /usr/doc/sup_commands
```

*At this point, Kazoo should be up and running on your EC2 instance. Now we will configure Kazoo for EC2 instance. We need to manually change some variables, especially IPs so that Kamailio can communicate effectively with freeSWITCH*

#### Configure the sip profiles in freeswitch 

```
# Install nano
yum install nano
```
```
# If you prefer vim then
yum install vim
```
*Then Edit the files as follows*

```
nano /etc/kazoo/freeswitch/sip_profiles/sipinterface_1.xml
```

```
# TODO :  Using a text editor of your choice (vim/nano) edit the sipinterface_1.xml file as follows**

 Set <param name="ext-rtp-ip" value="auto"/>   to 
       <param name="ext-rtp-ip" value="x.x.x.x."/> 
    (x.x.x.x is the external IP given by AWS EC2)

Set <param name="local-network-acl" value="localnet.auto"/>  to                
       <param name="local-network-acl" value="NOPE"/>
# “NOPE” doesn't matter, just not localnet.auto

**Setting the “ext-rtp-ip” to your public IP enables kamailio to configure FreeSWITCH
**By setting "local-network-acl" to “NOPE” ensures that FreeSWITCH sends the SIP packets to the right IP address
```

**Finally : Edit the local.cfg file** 
```
nano /etc/kazoo/kamailio/local.cfg

listen=UDP_SIP advertise x.x.x.x:5060 
listen=TCP_SIP advertise x.x.x.x:5060
(where x.x.x.x is your public AWS address.)

[This ensures that FreeSWITCH can send the ACK message to the right address 
over the right IP and avoids losing connection during calls]
```

**If Firewall was not installed**

```
yum install firewalld
systemctl enable firewalld
systemctl status firewalld
```











Source: PowerPBX
[Kazoo V4 Single Server Install](https://www.powerpbx.org/content/kazoo-v4-single-server-install-guide-v1)






