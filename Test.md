# Prerequisites

It is assumed the reader has an intermediate knowledge of PowerShell and Network Technologies. 

The examples in this chapter require an Active Directory domain environment with at least one Windows Server 2016 domain controller and one Windows Server 2016 member server. No prerequisites for forest and domain mode. Make sure Windows remote management (WinRM) is enabled on all computers.

# Introduction

The more I play with it, the more I like it: PowerShell and the networking Cmdlets. Everybody knows ping. Everyone knows arp. But only very few have dealt with advanced network tasks in PowerShell. I want to change that here and will give the reader many useful tips to demonstrate that the object-oriented PowerShell has a lot to offer when it comes to doing network tasks, monitoring, automation and troubleshooting. Why do I actually think it’s worth talking about PowerShell and Network Technology?

In over 20 years of experience in IT, I can't tell how many times I've heard those words: "Implementing a monitoring solution will take a lot of time and cost a lot. We can't afford that. Now we have no choice but to hope that everything works fine and no errors occur on our production servers." This means that in the event of errors and failures, there is a greatly delayed reaction and thus a longer downtime. But these people are right, implementing a centralized monitoring solution will mean much effort, but does it really have to be an enterprise solution? Can't we build a small monitoring solution in a few simple steps? In any case, anything is better than doing nothing. That`s what I believe.

The same goes to network security. "We have servers with a graphical user interface as we are used to. The large number of Windows updates are a little annoying, but that's just the way it is with Windows. We're used to it, it's always been that way." No, not anymore. With the introduction of Windows Server Core and Nano Server, the Windows world has changed. Server Core and Nano Server require fewer updates, less storage space, and are more secure than any other Windows operating system so far due to the smaller number of features installed. And all this means one thing: Get familiar with PowerShell.

This brings us to our topic for this chapter: PowerShell and network technology. With only a small set of Cmdlets in hand and some scripting know how we are able to do wonderful things at zero cost.
Let’s dive in.

# IP Configuration

With the introduction of the headless Nano Server and Server Core, PowerShell is becoming increasingly important. Previously, we did the IP configuration graphically or with the good old netsh command. Today PowerShell is here and allows us to configure IP address, subnet mask, default gateway and the DNS server settings more easily than ever before.

To configure an IPv4 Address you may want to check the Network Adapter Name or the Interface Index first.
```
PS C:\> Get-NetAdapter | Where-Object Status -EQ 'Up' | Format-Table -AutoSize
```
Now we know which network cards are up and we can move on with New-NetIPAddress to configure the network settings. Make sure to provide the Subnet Mask in Prefix Notation.
```
PS C:\> New-NetIPAddress -InterfaceIndex 4 -IPAddress 10.0.0.7 -PrefixLength 24 -DefaultGateway 10.0.0.1
```
Our next step is configuring the DNS Server and the alternate DNS Server.
```
PS C:\> Set-DnsClientServerAddress -InterfaceIndex 4 -ServerAddresses 127.0.0.1,10.0.0.8
```
To check the settings run Get-NetIPConfiguration.
{line-numbers=off}
```
PS C:\> Get-NetIPConfiguration

InterfaceAlias       : Ethernet 2
InterfaceIndex       : 4
InterfaceDescription : Microsoft Hyper-V Network Adapter #2
NetProfile.Name      : sid-500.com
IPv6Address          : 2001:aefb::1
IPv4Address          : 10.0.0.7
IPv6DefaultGateway   :
IPv4DefaultGateway   : 10.0.0.1
DNSServer            : ::1
                       127.0.0.1
                       10.0.0.8

PS C:\>
```
For the final test of the connectivity I like to use Test-NetConnection. Test-NetConnection without specifying a destination address attempts to reach a standard address on the Internet.
{line-numbers=off}
```
PS C:\> Test-NetConnection

ComputerName           : internetbeacon.msedge.net
RemoteAddress          : 13.107.4.52
InterfaceAlias         : Ethernet 2
SourceAddress          : 10.0.0.7
PingSucceeded          : True
PingReplyDetails (RTT) : 4 ms

PS C:\>
```
A computer makes its decisions about how to handle a packet based on its routing table. Either the computer manages the package by itself or it uses it's configured default gateway if it detects that the destination host is outside its own subnet. The default gateways IP Address is always associated with Prefix 0.0.0.0/0.
```
PS C:\> Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Select-Object -ExpandProperty NextHop
10.0.0.1
PS C:\>
```
Remember that doing it this way always gives us the correct IP of the Default Gateway, regardless of how many network cards are present and configured. It’s called Default Route (0.0.0.0/0) or Last Gateway of Resort. If we can easily read the default gateway this way, then it must also work with remote computers. Assuming we do not know AzServers Default Gateway IP-Address, we can get it straightforward with Invoke-Command and Get-NetRoute.
```
PS C:\> Invoke-Command -ComputerName AzServer01 -ScriptBlock {Get-NetRoute -DestinationPrefix 0.0.0.
0/0 | Select-Object -ExpandProperty NextHop}
10.0.0.1
```
Now we go one better! We can test if a remote computer Server02 has connectivity to its Default Gateway without knowing the IP Address of that Gateway.
```
PS C:\> Test-Connection -Source Server02 -ComputerName (Invoke-Command -Computer
Name Server02 -ScriptBlock {Get-NetRoute -DestinationPrefix '0.0.0.0/0' | Select
-Object -ExpandProperty NextHop})

Source        Destination     IPV4Address      IPV6Address
------        -----------     -----------      -----------
SERVER02      192.168.0.1
SERVER02      192.168.0.1
SERVER02      192.168.0.1
SERVER02      192.168.0.1

PS C:\>
```
This overview of what I think are the most important network cmdlets is a foretaste of what else we have in mind in this chapter.

# Discovering the network with PowerShell

Ping is without a doubt the number one network monitoring and troubleshooting tool, it's well known, it's fast and we can use it on almost all operating systems. It sends ICMP Echo Requests to test the connectivity to other hosts. There are now two more ascending stars that came with Windows PowerShell. One of them is called Test-Connection, the other Test-NetConnection. The main question for this part is how to gather information which Windows Server is up and running, which ports are open on your Servers and how to create a network IP overview of all Windows Servers. But first, let’s talk a little about the Cmdlets we need.

## Test-Connection

When I published my website article "The new version of ping: Test-Connection" I was really surprised how many in the IT-Pro community hadn't heard of this Cmdlet. Test-Connection uses the win32_ping status WMI object. Compared to ping Test-Connection returns more information. In addition to the source object, we can also collect information about the IPv6 Address. In my case it's an IPv6 Global Unicast Address, which is similar to an IPv4 Public Address. A nice view: All leading zeros in the IPv6 Address are automatically omitted.
```
PS C:\> ping AzServer01

Pinging azserver01.sid-500.com [10.0.0.4] with 32 bytes of data:
Reply from 10.0.0.4: bytes=32 time=1ms TTL=128
Reply from 10.0.0.4: bytes=32 time=1ms TTL=128
Reply from 10.0.0.4: bytes=32 time=1ms TTL=128
Reply from 10.0.0.4: bytes=32 time=1ms TTL=128

Ping statistics for 10.0.0.4:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 1ms, Maximum = 1ms, Average = 1ms
PS C:\>
PS C:\> Test-Connection AzServer01 | Format-Table -AutoSize

Source Destination IPV4Address IPV6Address  Bytes Time(ms)
------ ----------- ----------- -----------  ----- --------
AzDC01 AzServer01  10.0.0.4    2001:aefb::3 32    1
AzDC01 AzServer01  10.0.0.4    2001:aefb::3 32    0
AzDC01 AzServer01  10.0.0.4    2001:aefb::3 32    0
AzDC01 AzServer01  10.0.0.4    2001:aefb::3 32    1

PS C:\>
```
What I like best is the silent mode, which I like to call Shut-Up-Mode.
```
PS C:\> Test-Connection AzServer01 -Quiet
True
PS C:\>
```
Test-Connection supports sending ICMP Echo Requests to multiple destinations at once and specifying the number of requests.
```
PS C:\> Test-Connection AzServer01,AzDc01 -Count 1 | Format-Table -AutoSize

Source Destination IPV4Address IPV6Address                 Bytes Time(ms)
------ ----------- ----------- -----------                 ----- --------
AzDC01 AzServer01  10.0.0.4    2001:aefb::3                32    0
AzDC01 AzDc01      10.0.0.7    fe80::a456:9139:9b95:df7e%4 32    0

PS C:\>
```
With the good old ping command the source is always localhost. But the good news is: With Test-Connection you can specify a source computer. Note that I'm logged on AzDC01 and I want to perform a ping from computer AzServer01.
```
PS C:\> Test-Connection -Source AzDc02 -Destination AzServer01 | Format-Table -AutoSize

Source Destination IPV4Address IPV6Address  Bytes Time(ms)
------ ----------- ----------- -----------  ----- --------
AzDC02 AzServer01  10.0.0.4    2001:aefb::3 32    1
AzDC02 AzServer01  10.0.0.4    2001:aefb::3 32    0
AzDC02 AzServer01  10.0.0.4    2001:aefb::3 32    0
AzDC02 AzServer01  10.0.0.4    2001:aefb::3 32    1

PS C:\>
```
If it is possible for one source computer, then it must also work for several. As an example, we could try to test the ICMP connectivity of all domain joined Windows Servers.
```
PS C:\> Test-Connection -ComputerName ((Get-ADComputer -Filter 'operatingsystem -like "*server*"').N
ame) -Count 1 | Format-Table -AutoSize

Source Destination IPV4Address IPV6Address                 Bytes Time(ms)
------ ----------- ----------- -----------                 ----- --------
AzDC01 AzDC01      10.0.0.7    fe80::a456:9139:9b95:df7e%4 32    0
AzDC01 AzServer01  10.0.0.4    2001:aefb::3                32    0
AzDC01 AzDC02      10.0.0.8                                32    0

PS C:\>
```
We owe all this and more to the object-orientated Powershell. I like it! As shown, with PowerShell it’s easy to do some network checks in an Active Directory Domain. That’s because all computer accounts are stored in the Active Directory database. But are they really up? Grabbing those computer accounts don’t mean that they are switched on and users are logged on to them.

## Is this host up and who is logged in?

Is ping a reliable way to check if a host is up? Opinions differ. 

The decisive factor for testing IP connectivity in Windows networks is the host based Windows Firewall. If both computers are on the same subnet, the ping may fail, but the host may be up. That’s because Windows Firewall may block ICMP requests. So far so good. But how can we determine if a host is up or not? I have a tailored solution for this problem, we simply check the ARP cache. If the ping fails, but the ARP request was successful, then it is pretty sure that the host is up! Note, that this applies only to computers that are in the same subnet. With a  small function in PowerShell, we see that 10.0.0.4 is up, but curiously the ping failed.
```
"
$IPAddress=Read-Host "Enter IP Address"
arp -d
$ping=Test-Connection -ComputerName $IPAddress -Count 1 -Quiet
$arp=[boolean](arp -a | Select-String "$IPAddress")
If (!$ping -and $arp) {
Write-Host "ICMP: failure, ARP: successful. Possible Cause on ${IPAddress}: Windows Firewall is blocking traffic"
}

Enter IP Address: 10.0.0.4
ICMP: failure, ARP: successful. Possible Cause on 10.0.0.4: Windows Firewall is blocking traffic

PS C:\> 
```
This means, we could use Test-Connection to check multiple computers if they’re up before doing any further actions on them. 
With that on mind, I wrote a nice little script which first tests with Test-Connection if the remote computer is up, then shows the logged on users (disconnected users are not displayed) and then sends a message to these users session over the network.
```
$cname=Read-Host "Enter Computername"
$test=Test-Connection $cname -Count 1 -ErrorAction SilentlyContinue
$result=@()
If ($test) {
       $message=Read-Host 'Enter message'
       Invoke-Command -ComputerName $cname -ScriptBlock {quser} | Select-Object -Skip 1 | Foreach-Object {
       $b=$_.trim() -replace '\s+',' ' -replace '>','' -split '\s'
       $result+= New-Object -TypeName PSObject -Property ([ordered]@{
                'User' = $b[0]
                'Computer' = $cname
                'Session' = $b[1]
                'Date' = $b[5]
                'Time' = $b[6..7] -join ' '
                })      
                } 
        $result | Format-Table
        Invoke-Command -ComputerName $cname -ScriptBlock {msg * /V $using:message} -ErrorAction SilentlyContinue
           }
else {
        Write-Host "Failed to connect to $cname" -ForegroundColor Red
        throw 'Error'
     }
     
Enter Computername: AzServer01
Enter message: Server shutdown in about 10 minutes.

User    Computer   Session   Date      Time   
----    --------   -------   ----      ----   
patrick AzServer01 rdp-tcp#2 5/16/2018 7:26 PM

Sending message to session Console, display time 60
Async message sent to session Console
Sending message to session RDP-Tcp#2, display time 60
Async message sent to session RDP-Tcp#2

PS C:\> 
     
```
IMAGE ??????

What have we done so far? We checked if the computer responds to ICMP requests. If so, we use quser to query the logged on users. If not, the script throws an error. As mentioned, we retrieve the computer names from the Active Directory database. Sometimes, however, there is a need to get more information than the computer name. Then we may need a complete network configuration overview of all Windows Servers. Let’s do it.


