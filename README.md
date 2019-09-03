Hybrid Cloud Infrastructure Using OpenVPN
=========================================
Hybrid cloud means the use of both public cloud platform and on-premises resources. Public cloud platforms, such as Amazon Web Services, Microsoft Azure or Google Cloud platform which offers infrastructure as a service (IaaS). A hybrid cloud enables an organization to extend their datacenter capacity, utilize new cloud-native capabilities, move applications closer to customers, and create a backup and disaster recovery solution with cost-effective high availability. Hybrid cloud has benefits but comes with technical, business and management challenges and requires the expertise of cloud architects.

Background
==========
I was asked to automate and orchestrate on-demand Hybrid Cloud Infrastructure which will help us to connect with our client's private environment to identify loopholes and perform necessary actions requires in their environment.

Problem
=======
How to create a Hybrid Environment?

Solution
========
AWS and OpenVPN to the rescue. In this document I am using AWS environment to build my public infrastructure but we can also Azure, GCP or any other cloud infrastructure provider.

Create Certificate Authority
============================
We are not going to cover building a certificate authority in this post. We have to build CA and Diffie Hellman Key and then signed our server and client certificates using the CA to be use by OpenVPN. However, we can easily achieve this by using easy-rsa, please follow [Setting up your own Certificate Authority](https://openvpn.net/community-resources/setting-up-your-own-certificate-authority-ca/).

Install OpenVPN -  Server (AWS Linux 2 AMI)
===========================================
<pre>
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum -y install yum-utils
yum-config-manager --enable epel
yum -y install openvpn
</pre>

Server Configurations - server.conf
===================================
-  server.conf
	<pre>
	port 443
	proto tcp
	dev tun
	ca ca.crt
	cert server.crt
	key server.key
	dh dh.pem
	topology subnet
	server 162.200.100.0 255.255.255.0
	route [Your Client Subnet] [Subnet Mask]
	push "route [Your Server Private Subnet] [Subnet Mask]"
	client-config-dir ccd
	ifconfig-pool-persist /var/log/openvpn/ipp.txt
	keepalive 10 120
	cipher AES-256-CBC
	persist-key
	persist-tun
	status /var/log/openvpn/openvpn-status.log
	verb 3
	</pre>

- ccd/client
	<pre>
 	iroute [Your Client Subnet] [Subnet Mask]
 	</pre>

Client Configurations - client.conf 
===================================
-  client.conf
	<pre>
	client
	dev tun
	proto tcp
	remote [Your Server Public IP Address]
	port 443
	topology subnet
	resolv-retry infinite
	nobind
	persist-key
	persist-tun
	ca ca.crt
	cert client.crt
	key client.key
	remote-cert-tls server
	cipher AES-256-CBC
	verb 3
	</pre>

Firewalls - Server
==================
- iptables   (IPV4)
	<pre>
	iptables -A FORWARD -i eth+ -o tun+ -j ACCEPT
	iptables -A FORWARD -i tun+ -o eth+ -m state --state RELATED,ESTABLISHED -j ACCEPT
	</pre>

Enable IPv4 Forwarding - changes to /etc/sysctl.conf
====================================================
<pre>
net.ipv4.ip_forward = 1
</pre>

What about IPV6? [Optional but Interesting]
===========================================
You can create a IPv6 tunnel over IPv4 with OpenVPN. Add the following to your server.conf file and there will no changes require to your client.conf file.

 -  Server Configurations
 	<pre>
 	tun-ipv6
	push tun-ipv6
 	server-ipv6 2001:0db8:ee00:abcd::/64
 	route-ipv6 [Your Client Ipv6 Subnet CIDR]
 	push "route-ipv6 [Your Server Ipv6 CIDR]"
 	</pre>

 - ccd/client
	<pre>
	iroute-ipv6 [Your Client Ipv6 Subnet CIDR]
	</pre>

- ip6tables (IPV6)
	<pre>
	ip6tables -A FORWARD -i eth+ -o tun+ -j ACCEPT
	ip6tables -A FORWARD -i tun+ -o eth+ -m state --state RELATED,ESTABLISHED -j ACCEPT
	</pre>

- Enable Ipv6 forwarding changes to /etc/sysctl.conf
	<pre>
	net.ipv6.conf.default.forwarding = 1
	net.ipv6.conf.all.forwarding = 1
	net.ipv6.conf.eth0.forwarding = 0
	net.ipv6.conf.eth0.accept_ra = 1
	net.ipv6.conf.all.accept_ra = 1
	net.ipv6.conf.default.accept_ra = 1
	</pre>

Discussion
==========
This will provide you an overview to understand and implement a hybrid solution using AWS and OpenVPN.

Have you implemented this solution?

Yes, I implemented using serverless architecture which allows us to automate and orchestrate OpenVPN environment and connect with our clients.

Is this cost-effective?

Yes, It's cheaper than building a hybrid environment using AWS VPN Gateway and AWS Direct Connect.
