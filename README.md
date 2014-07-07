WFS
===

Wan Failover Script

Original script by louwrentius@gmail.com

Modified to add a cleanup function to remove routes added by the script on startup.
The cleanup function (called wfs_stop) only works with deamon mode.


Design
======

WFS checks if your WAN connection is still up by sending ICMP messages to multiple target
hosts. It automatically switches between primary and secondary WAN link by changing the
default gateway.

## Installation

 * untar the tar file
 * run the install.sh script.
 * edit the configuration /etc/wfs/wfs.conf to your liking
 * add target hosts used for testing in /etc/wfs/targets.txt
 * start wfs through "/etc/init.d/wfs start"

## For email notifications (Ubuntu)

  * apt-get install mailutils postfix (installs mail and a local MTA)
  * configure postfix as an internet gateway
  * do not forward an inbound mail to this instance, it should be behind a firewall and for outbound notification traffic only.

## Assumption

I asume that your host has access to two different routers that act as the primary and backup
gateway for the primary WAN connection and backup WAN connection. It may be that your host
has two interfaces, with two IP-addresses for each WAN connection, but this is not required.

## How WFS tests for link availability

WFS uses ICMP messages to test if the primary WAN link is still up. There is a risk that the
target host goes off-line while the WAN link is perfectly fine. This would cause an
unnecessary fail-over event. To prevent unnecessary failover, at least two hosts should be
used for availability testing.

WFS can use an arbitrary number of target hosts for availability testing. WFS tests each host
in sequence. Thus, first host 1 is tested, then host 2, until the last host, after which WFS
starts again with the first.

It is strongly advised to use more than two target hosts. WFS sends an ICMP message
each specified interval, and this could be considered abusive by the target hosts. To
increase the delay between tests, add more hosts to the 'targets' file /etc/wfs/targets.txt
and/or increase the test delay between hosts.

Your default gateway address on your ISP's network is a good host to have in your targets.txt
file as this confines some of the ICMP traffic to your subnet at your ISP as well.  It is
also a great test of whether you can reach the Internet or not!.

## How failover works

WFS changes the default gateway and thus the default route to which all traffic is sent. It
uses the native 'route' command found on most Linux distributions.

## When the primary WAN link is restored

When executed, WFS adds static routes for all target hosts that are used for availability
monitoring. These routes specifically use the router/gateway for the primary WAN interface,
even if the default gateway is switched to the failover connection.

This allows WFS to monitor if the primary WAN interface is still down or back up again.
If the latter is true, then WFS will switch the default route back to the primary interface.

Example:

The primary WAN gateway is 10.0.0.1 in this example.

    server:~# route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    74.125.77.104   10.0.0.1        255.255.255.255 UGH   0      0        0 eth1
    209.85.229.104  10.0.0.1        255.255.255.255 UGH   0      0        0 eth1
    219.25.29.14  10.0.0.1        255.255.255.255 UGH   0      0        0 eth1
    192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
    10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth1
    0.0.0.0         10.0.0.1        0.0.0.0         UG    0      0        0 eth1

When a failover to a backup WAN link occurs (192.168.0.1) it would look like this:

    server:~# route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    74.125.77.104   10.0.0.1        255.255.255.255 UGH   0      0        0 eth1
    209.85.229.104  10.0.0.1        255.255.255.255 UGH   0      0        0 eth1
    219.25.29.14  10.0.0.1        255.255.255.255 UGH   0      0        0 eth1
    192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
    10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth1
    0.0.0.0         192.168.0.1     0.0.0.0         UG    0      0        0 eth0

## Configuration options

To configure WFS to your needs, edit the WFS script itself or edit /etc/wfs/wfs.conf.

The first thing you should do is to edit the file containing IP addresses of hosts that will
be used for testing.

Contents of /etc/wfs/targets.txt:

    74.125.77.104
    209.85.229.104
    219.25.29.14

These are the hosts that are used to test if the primary WAN interface is available. Please
note that this is an example.

PRIMARY_GW=10.0.0.1
SECONDARY_GW=192.168.0.1

Here, the primary and secondary (failover) WAN gateway adress is configured. When a failure
is detected, WFS will switch from the primary gateway to the secondary gateway.

INTERVAL=20
TEST_COUNT=2

WFS will test the availability of the primary WAN link each 'INTERVAL' seconds, in this
example thus 20 seconds. WFS will test by sending 2 ICMP messages.

THRESHOLD=3

To prevent a premature failover, such as when connectivity is only lost for a couple of
seconds, or when a single test target goes down, a threshold must be reached before a
failover is performed. In this example, the failover test must fail three times before
a failover is committed. Based on the interval used in the previous example, a failover will
take about a minute after the connection has gone bad. By changing the values, you can
switch faster or slower.

COOLDOWNDELAY=120

To prevent fast switching between primary and secondary WAN link due to a flaky connection,
a cool down period is used. Only after this period WFS will switch back to the primary
interface again.

From 2.03 onwards COOLDOWNDELAY is deprecated and replaced with:

COOLDOWNDELAY01=3600
COOLDOWNDELAY02=600

COOLDOWNDELAY01 is the cool down interval after a failover event.  All testing will pause
and connectivity remain through the secondary WAN until this counter has expired.  Value
is in seconds.

COOLDOWNDELAY02 is the cool down interval on failback event.  All testing will pause and
connectivity remain through the primary WAN until this counter has expired.  Value is
in seconds.

Sometimes you may want to have a long cooldown period after a failover event to keep services
stable while you troubleshoot the primary wan connection.  Whereas after the connection
fails back to the primary you would want a shorter period before connection monitoring
starts again

MAIL_TARGET=""

Assuming that your backup WAN link allows you to send email messages, you can specify an
email address. This address will receive a message when a failover occurs or when the primary
link is restored.  Note this email functionality requires the command line mail command.
This can be installed by apt-get install mailutils on Ubuntu.  An MTA such as postfix is also
required.

The email will contain a copy of the routing table as it is after the failover or failback
event.  This allows you to verify that the route has changed and is correct.

PRIMARY_CMD=""
SECONDARY_CMD=""

These two variables can be used to execute a custom command after a failover or restore is
performed. When switching to the secondary interface, the secondary_cmd command is executed.
When switching back to the primary interface, primary_cmd is executed.

MAX_LATENCY=1

The maximum allowed latency on an ICMP ping request before the request is considered to have
failed.  This allows us to failover in the even of a severely degraded connection which is
still live but running very slowly.

## Logging

Log messages are sent to /var/log/daemon or /var/log/messages.

Logging can be more verbose by configuring the 'DEBUG' option within the configuration file.

Example of debug output:

    May 29 21:47:08 server WFS: INFO ------------------------------
    May 29 21:47:08 server WFS: INFO  WAN Failover Script 2.0
    May 29 21:47:08 server WFS: INFO ------------------------------
    May 29 21:47:08 server WFS: INFO  Primary gateway: 10.0.0.1
    May 29 21:47:08 server WFS: INFO  Secondary gateway: 10.0.0.1
    May 29 21:47:08 server WFS: INFO  Threshold before failover: 3
    May 29 21:47:08 server WFS: INFO  Number of target hosts: 4
    May 29 21:47:08 server WFS: INFO  Tests per host: 2
    May 29 21:47:08 server WFS: INFO ------------------------------
    May 29 21:47:08 server WFS: INFO Starting monitoring of WAN link.
    May 29 21:47:09 server WFS: INFO WAN Link: PRIMARY
    May 29 21:47:30 server WFS: INFO Host 74.125.77.104 UNREACHABLE
    May 29 21:47:30 server WFS: INFO Failed targets is 1, threshold is 3.
    May 29 21:47:30 server WFS: INFO WAN Link: PRIMARY
    May 29 21:47:36 server WFS: INFO Host 69.147.125.65 UNREACHABLE
    May 29 21:47:36 server WFS: INFO Failed targets is 2, threshold is 3.
    May 29 21:47:36 server WFS: INFO WAN Link: PRIMARY
    May 29 21:47:42 server WFS: INFO Host 98.137.149.56 UNREACHABLE
    May 29 21:47:42 server WFS: INFO Failed targets is 3, threshold is 3.
    May 29 21:47:42 server WFS: INFO Primary WAN link failed. Switched to secondary link.
    May 29 21:48:18 server WFS: INFO Host 209.85.227.105 UNREACHABLE
    May 29 21:48:18 server WFS: INFO Failed targets is 3, threshold is 3.
    May 29 21:48:24 server WFS: INFO Host 74.125.77.104 UNREACHABLE
    May 29 21:48:24 server WFS: INFO Failed targets is 3, threshold is 3.
    May 29 21:48:30 server WFS: INFO Host 69.147.125.65 UNREACHABLE
    May 29 21:48:30 server WFS: INFO Failed targets is 3, threshold is 3.
    May 29 21:48:36 server WFS: INFO Host 98.137.149.56 UNREACHABLE
    May 29 21:48:36 server WFS: INFO Failed targets is 3, threshold is 3.
    May 29 21:48:42 server WFS: INFO Host 209.85.227.105 UNREACHABLE
    May 29 21:48:42 server WFS: INFO Failed targets is 3, threshold is 3.
    May 29 21:48:48 server WFS: INFO Host 74.125.77.104 UNREACHABLE
    May 29 21:48:48 server WFS: INFO Failed targets is 3, threshold is 3.
    May 29 21:48:54 server WFS: INFO Failed targets is 2, threshold is 3.
    May 29 21:49:01 server WFS: INFO Failed targets is 1, threshold is 3.
    May 29 21:49:07 server WFS: INFO Primary WAN link OK. Switched back to primary link.
    May 29 21:49:43 server WFS: INFO WAN Link: PRIMARY
    May 29 21:50:04 server WFS: INFO WAN Link: PRIMARY
    May 29 21:50:25 server WFS: INFO WAN Link: PRIMARY

