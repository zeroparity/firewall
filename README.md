# FreeBSD 10 firewall via pf on the soekris net6501-70

In 2014 I decided to replace a Draytek Vigor all in one DSL router / wireless access point which had served us well for a couple of years, with something more configurable. After doing some research I made the decision to purchase a soekris net6501-70 and a small Intel ssd. Onto this I installed FreeBSD 10. As I'm stuck on a slow DSL connection, I also acquired a D-Link DSL-320B ADSL modem.

Reviewing this setup today, it would seem to make more sense to virtualise the firewall on a home server. However, when I set this up - there was a requirement for the firewall machine to be physically located in a shoe cupboard right next to the telecoms NTE socket. The soekris was relatively expensive for such a low powered box, but has proven to be extremely stable. I would recommend soekris without hesitation.

## Requirements
* Be completely stable. Spouse acceptance factor was top of the list of priorities. 
* Route traffic from our home network to the Internet over our Batelco ADSL connection.
* Firewall our Internet connection.
  * Don’t respond to icmp echo requests on the external interface.
  * No open ports on the external interface (except for SSH which is only available inbound from my VPS IP address)
  * Drop any traffic outbound that shouldn’t be going onto the Internet (SMB traffic, traffic heading for ports 4444, 31337 etc.)
* Provide DHCP service to our home network
* Run a caching DNS service - and forward DNS requests to OpenDNS (I'm using dnsmasq for DHCP / DNS)
* Connect via OpenVPN to my VPS
  * Selectively route traffic from designated hosts on my home network via the VPN (useful for the Amazon FireTV to appear as if we are in the US)
  * Forward port 3128 to the Squid proxy running on my VPS over the VPN.

## Why FreeBSD?
* I've been using it on and off since version 3.4
* It's a capable operating system with a huge package and ports collection.
* It's predictable and well documented.
* OpenBSD would also work really well, but hey ho. FreeBSD it was.

## Why the soekris net6501-70?
* 6501-70 was the fastest soekris device available at the time of purchase.
* Its silent. Used with an SSD it has no moving parts. 
* Uses very little power.
* Can be managed properly via a serial connection.
* Tolerates being hidden in a warm cupboard without airflow.
* Quirks seem to be well understood. 
* It's completely stable once setup.

## Caveats
* Due to lack of ACPI support in the soekris BIOS I used the i386 FreeBSD build.
* Getting FreeBSD installed required modifying the installation image to enable the console on serial port.
* Due to some kernel modifications, I have to rebuild the kernel from source after each freebsd-update.
* The atom processor in the soekris is slow. Kernel builds take a long time relative to a more capable box (I could solve this by compiling on another machine, which would arguably be a better approach than on the firewall itself).

## Firewall setup
* The pf documentation / examples at calomel.org really helped.
* The soekris net6501 comes with 4 Intel NICs on board (its easy to upgrade to 8 or 12). I'm using the first for the DSL modem (which FreeBSD calls em0), another for my home network, and the third is connected to an Intel NUC running ESXi which is isolated from the home network and only has Internet access. This DMZ approach provides a fairly safe DMZ for examining malware etc.


## ppp.conf

FreeBSD runs PPPD in user-land. This can bottleneck throughput on fast network connections.
Sadly my network is not fast enough to be bottlenecked by this (10Mb down / 2Mb up ADSL).
If I ever get a fast Internet connection I will switch to MPD which runs in kernel mode.

```
default:
  set log Phase TUN Command Warning
  disable iface-alias
  set timeout 0
  set ifaddr 10.0.0.1/0 10.0.0.2/0 0.0.0.0 0.0.0.0

batelco_adsl:
  set device PPPoE:em0
  set speed sync
  set mru 1492
  set mtu 1492
  set authname <username>
  set authkey <password>
  set dial
  set login
  add default HISADDR
```

Note that before I added the "disable iface-alias" directive, every time my Internet address changed (once per day with Batelco) the old IP address would remain as an alias against the tun0 interface. As the device was running 24x7 the number of aliases would grow and grow. It took me a while to realise what was going on, but adding that line fixed the issue.

## pf.conf

These are some select excerpts from my pf.conf. calomel.org provided the base of my configuration and they provide a way better explanation that I could.

I have a CentOS VPS hosted in the US that runs OpenVPN, I have ipchains rules on the VPS machine to allow it to IP masquerade Internet access to clients coming in over the VPN.
I have a few hosts in my home network that I want to route all traffic exiting my network over the VPN. This is useful for geographically restricted services such as Amazon FireTV, Amazon Echo etc.

To do this in pf.conf
  * Create a table with the addresses we want to route over the VPN 

  ```
   table <HostsToRouteViaVPN> {192.168.0.10, 192.168.0.11, 192.168.0.12}
  ```

  * Inbound rules (from my home network)

  ```
   pass in log on $IntIf route-to ($VpnIf $VpnGW) inet proto tcp from <HostsToRouteViaVPN> to any $TcpState $OpenSTO
   pass in log on $IntIf route-to ($VpnIf $VpnGW) inet proto udp from <HostsToRouteViaVPN> to any $UdpState $OpenSTO
   pass in log on $IntIf route-to ($VpnIf $VpnGW) inet proto icmp from <HostsToRouteViaVPN> to any $IcmpPing $UdpState $OpenSTO
  ```

  * Outbound rules (from my firewall to the VPS)

  ```
  pass out log on $VpnIf inet proto tcp from ($VpnIf) to <HostsToRouteViaVPN> $TcpState
  pass out log on $VpnIf inet proto udp from ($VpnIf) to <HostsToRouteViaVPN> $UdpState
  pass out log on $VpnIf inet proto icmp from ($VpnIf) to <HostsToRouteViaVPN> $UdpState
  ```

I also run a squid proxy on my VPS. It's locked down so it can be accessed only from the loopback and the OpenVPN interfaces.

In Bahrain a transparent proxy is used to filter access to adult and politically sensitive web sites. Unfortunately, this has a tendency to break some legitimate services that run over http. An example being linux package management tools such as apt-get and yum.

Using the following rule I can make it appear that the squid proxy is available on my home network, in reality we are using the proxy service on the VPS and our traffic is being tunneled over the VPN to the proxy. I can point my apt-cacher-ng server at this proxy and suddenly everything works again.

```
   $SquidPort = "3128"
   $SquidServer = <the OpenVPN interface IP address on my VPS>
   rdr on $IntIf inet proto tcp from !$IntIf to $IntIf port $SquidPort -> $SquidServer
```