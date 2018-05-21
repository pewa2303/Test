# Prerequisites

It is assumed the reader has an intermediate knowledge of PowerShell and Network Technologies. 

The examples in this chapter require an Active Directory domain environment with at least one Windows Server 2016 domain controller and one Windows Server 2016 that acts as a member server. No prerequisites to the forest and domain mode. Make sure Windows remote management (WinRM) is enabled on all computers.

# Introduction

Humans are creatures of habit. Once you have learned something, you usually do it in exactly the same way. Even if something new should come, you have confidence in what you are used to. I want to change this. With you. It's about change, it's about looking at something new, because I'm convinced that much of my chapter will be new to you. It is about network commands and PowerShell. Not ping and tracert, PowerShell Cmdlets that can replace, or at least complement your old habits. 
Because I love them, and I hope you'll love them too.

Let me lose some thoughts about the topic and my experience before we get to the technical part ...

The more I play with it, the more I like it: PowerShell and the networking Cmdlets. Everybody knows ping. Everyone knows arp. But only very few have dealt with advanced network tasks in PowerShell. I want to change that here and will give the reader many useful tips to demonstrate that the object-oriented PowerShell has a lot to offer when it comes to doing network tasks, monitoring, automation and troubleshooting. Why do I actually think it’s worth talking about PowerShell and Network Technology?

In over 20 years of experience in IT, I can't tell how many times I've heard those words: "Implementing a monitoring solution will take a lot of time and cost a lot. We can't afford that. Now we have no choice but to hope that everything works fine and no errors occur on our production servers." This means that in the event of errors and failures, there is a greatly delayed reaction and thus a longer downtime. But these people are right, implementing a centralized monitoring solution will mean much effort, but does it really have to be an extensive solution? Can't we build a small monitoring solution in a few simple steps? In any case, anything is better than doing nothing. That`s what I believe.

The same goes to network security. "We have servers with a graphical user interface as we are used to. The large number of Windows updates are a little annoying, but that's just the way it is with Windows. We're used to it, it's always been that way." No, not anymore. With the introduction of Windows Server Core and Nano Server, the Windows world has changed. Server Core and Nano Server require fewer updates, less storage space, and are more secure than any other Windows operating system so far due to the smaller number of features installed. Should I really believe that Microsoft will not change the current Windows Server strategy? No, not me. Small systems are the future. Who needs a web server with a 12 GB operating system? Much of the software of this operating system is not needed at all. Remember when you install a Windows Server Server Core is the default installation option. That's a very clear signal from Microsoft. All this means one thing: The GUI is fading away and it's time to get familiar with PowerShell.

This brings us to our topic for this chapter: PowerShell and network technology. With only a small set of Cmdlets in hand and some scripting know-how we are able to do amazing things at zero cost. You and me. Now. Let’s dive in.

# IP Configuration

With the introduction of the headless Nano Server and Server Core, PowerShell is becoming increasingly important. Previously, we did the IP configuration graphically or with the good old netsh command. Today PowerShell is here and allows us to configure IP Address, Subnet Mask, Default Gateway and the DNS server settings more easily than ever before. Let's do some basics at the beginning.

To configure an IPv4 Address with Windows PowerShell you may want to check the Network Adapter Name or the Interface Index first. Above all we are interested in network cards that are up.
```
PS C:\> Get-NetAdapter | Where-Object Status -EQ 'Up' | Format-Table -AutoSize
```
Now we can move on with New-NetIPAddress to configure the network settings. Make sure to provide the Subnet Mask in **Prefix Notation**. This completes the IP Configuration.
```
PS C:\> New-NetIPAddress -InterfaceIndex 4 -IPAddress 10.0.0.7 -PrefixLength 24 -DefaultGateway 10.0.0.1
```
Our next step is configuring the DNS Server and the alternate DNS Server.
```
PS C:\> Set-DnsClientServerAddress -InterfaceIndex 4 -ServerAddresses 127.0.0.1,10.0.0.8
```
I always check all settings. No matter how many times I've made the task before. To check all settings run Get-NetIPConfiguration.
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
For the final connectivity test I like to use Test-NetConnection. Test-NetConnection without specifying a destination address attempts to reach a standard address on the Internet. I can then see that my default gateway must be configured correctly.
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
Something else worth mentioning about the default gateway is that a computer makes its decisions about how to handle a packet based on its routing table. Either the computer manages the package by itself or it uses it's configured default gateway if it detects that the destination host is outside its own subnet. The default gateways IP Address is always associated with Prefix 0.0.0.0/0. With Get-NetRoute we can catch this PowerShell object property.
```
PS C:\> Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Select-Object -ExpandProperty NextHop
10.0.0.1
PS C:\>
```
Remember that gettting the IP Address of the Default Gateway with the **prefix 0.0.0.0/0 always gives us the correct IP of the Default Gateway**, regardless of how many network cards are present and configured. It’s called Default Route (0.0.0.0/0) or Last Gateway of Resort. If we can easily read the default gateway this way, then it must also work with remote computers. Assuming we do not know AzServers Default Gateway IP-Address, we can get it straightforward with Invoke-Command and Get-NetRoute.
```
PS C:\> Invoke-Command -ComputerName AzServer01 -ScriptBlock {Get-NetRoute -DestinationPrefix 0.0.0.0/0 | 
Select-Object -ExpandProperty NextHop}
10.0.0.1
```
Now we go one better! We can test if a remote computer Server02 has connectivity to its Default Gateway without knowing the IP Address of that Gateway, because the Default Gateways IP Address is always associated with the Default Route.
```
PS C:\> Test-Connection -Source Server02
-ComputerName (Invoke-Command -ComputerName Server02 -ScriptBlock {Get-NetRoute -DestinationPrefix '0.0.0.0/0' | 
Select-Object -ExpandProperty NextHop})

Source        Destination     IPV4Address      IPV6Address
------        -----------     -----------      -----------
SERVER02      192.168.0.1
SERVER02      192.168.0.1
SERVER02      192.168.0.1
SERVER02      192.168.0.1

PS C:\>
```
Yes, Server02 can reach it's Default Gateway.
This overview of what I think are the most important network cmdlets is a foretaste of what else we have in mind in this chapter. Let's move on with discovering our Windows Network.

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
PS C:\> Test-Connection -Destination AzServer01 | Format-Table -AutoSize

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
PS C:\> Test-Connection -Destination AzServer01 -Quiet
True
PS C:\>
```
Test-Connection supports sending ICMP Echo Requests to multiple destinations at once and specifying the number of requests.
```
PS C:\> Test-Connection -Destination AzServer01,AzDc01 -Count 1 | Format-Table -AutoSize

Source Destination IPV4Address IPV6Address                 Bytes Time(ms)
------ ----------- ----------- -----------                 ----- --------
AzDC01 AzServer01  10.0.0.4    2001:aefb::3                32    0
AzDC01 AzDc01      10.0.0.7    fe80::a456:9139:9b95:df7e%4 32    0

PS C:\>
```
With the good old ping command the source is always localhost. But the good news is: With Test-Connection you can specify a **source computer**. Note that I'm logged on AzDC01 and want to perform a ping from computer AzDC02 to AzServer01. It's a one liner, like many other PowerShell tasks.
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
Let's think ahead. If it is possible for one source computer, then it must also work for several. As an example, we could try to test the ICMP connectivity of all domain joined Windows Servers. Note, that this is a one liner again.
```
PS C:\> Test-Connection -ComputerName ((Get-ADComputer -Filter 'operatingsystem -like "*server*"').Name) -Count 1 | 
Format-Table -AutoSize

Source Destination IPV4Address IPV6Address                 Bytes Time(ms)
------ ----------- ----------- -----------                 ----- --------
AzDC01 AzDC01      10.0.0.7    fe80::a456:9139:9b95:df7e%4 32    0
AzDC01 AzServer01  10.0.0.4    2001:aefb::3                32    0
AzDC01 AzDC02      10.0.0.8                                32    0

PS C:\>
```
We owe all this and more to the object-orientated Powershell. I like it! As shown, with PowerShell it’s easy to do some network checks in an Active Directory Domain. That’s because all computer accounts are stored in the Active Directory database. But are they really up? Grabbing those computer accounts don’t mean that they are switched on and users are logged on to them. Which brings us to a question.

## Is this host up and who is logged in?

Is ping a reliable way to check if a host is up? Opinions differ. But it's definitely a good lead.

The decisive factor for testing IP connectivity in Windows networks is the host based **Windows Firewall**. If both computers are on the same subnet, the ping may fail, but the host may be up. That’s because Windows Firewall may block ICMP Requests. So far so good. But how can we determine if a host is up or not? I have a tailored solution for this problem, we simply check the ARP cache. If the ping fails, but the ARP request was successful, then it is pretty sure that the host is up! Note, that this applies only to computers that are in the same subnet. With a  small function in PowerShell, we see that 10.0.0.4 is up, but curiously the ping failed.
```
$IPAddress=Read-Host -Prompt "Enter IP Address"
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
Normally ping is allowed in domain networks and this is also the default setting of the Windows Firewall. That's what we're assuming now.
With that on mind, I wrote a nice little script which first tests with Test-Connection if the remote computer is up, then shows the logged on users (disconnected users are not displayed) and then sends a message to these users session over the network.
```
$cname=Read-Host -Prompt "Enter Computername"
$test=Test-Connection -Destination $cname -Count 1 -ErrorAction SilentlyContinue
$result=@()
If ($test) {
       $message=Read-Host -Prompt 'Enter message'
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
        Write-Host "Failed to connect to $cname"
        throw 'Error'
     }
```
After running the code you'll be asked to provide a computer name. Then enter the message that is send to all logged on users.
```
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
What have we done so far? We checked if the computer responds to ICMP requests. If so, we use quser to query the logged on users. If not, the script throws an error. As mentioned, we retrieve the computer names from the Active Directory database. Sometimes, however, there is a need to get more information than the computer name. Then we may need a complete network ip overview of all Windows Servers.

## Windows Server Network Topology

Our goal for this section is to make a network topology. We need all relevant network information of all domain joined Windows Server.

With all this information like reading the gateway and the IP configuration, it is easy to get a complete IP overview of all Windows Server systems. The most important values are retrieved, such as IP address, gateway, DNS server and more.
```
$getc=(Get-ADComputer -Filter 'operatingsystem -like "*server*"').Name
$test=Test-Connection -Destination $getc -Count 1 -ErrorAction SilentlyContinue
$reach=$test | Select-Object -ExpandProperty Address
$result=@()

foreach ($c in $reach)

{
$i=Invoke-Command -ComputerName $c -ScriptBlock {
        
            Get-NetIPConfiguration | 
            Select-Object -Property InterfaceAlias,InterfaceIndex,Ipv4Address,Ipv6Address,DNSServer
            Get-NetRoute -DestinationPrefix '0.0.0.0/0' | Select-Object -ExpandProperty NextHop}

            $result +=New-Object -TypeName PSCustomObject -Property ([ordered]@{

                'Server'= $c
                'Interface' = $i.InterfaceAlias -join "`r`n"
                'Index' = $i.InterfaceIndex -join "`r`n"
                'IPv4Address' = $i.Ipv4Address.IPAddress -join "`r`n"
                'IPv6Address' = $i.Ipv6Address.IPAddress -join ','
                'DNS Server' = ($i.DNSServer | Select-Object -ExpandProperty ServerAddresses) -join "`r`n"
                'Default Gateway' = $i | Select-Object -Last 1


                })
                
}
$result | Format-Table -AutoSize -Wrap
```
Running this code creates an PowerShell object that shows all relevant information about your Windows Servers. Putting all together in a function Get-ServerIPInfo and saving it in C:\Program Files\WindowsPowerShell\Modules enables you and all other users of your system to quickly re-use the code.
```
PS C:\> Get-ServerIPInfo

Server     Interface              Index IPv4Address        IPv6Address  DNS Server               Default Gateway
------     ---------              ----- -----------        -----------  ----------               ---------------
AzDC01     Ethernet 2             4     10.0.0.7           2001:aefb::1 ::1                      10.0.0.1       
                                                                        127.0.0.1                               
                                                                        10.0.0.8                                
AzServer01 Ethernet 5             5     10.0.0.9           2001:aefb::3 168.63.129.16            10.0.0.1       
           Ethernet 4             8     10.0.0.4                        10.0.0.7                                
AzDC02     Ethernet 3             5     10.0.0.8                        ::1                      10.0.0.1       
                                                                        10.0.0.7                                
                                                                        127.0.0.1                               


PS C:\> 
```
Of course, this is not only possible for Windows servers, but also for Domain Controllers or Windows Clients. But here you have to consider if this makes sense, because Windows clients are usually not always switched on. Remember that the code initiates a live query over all given systems. If one of the computer is switched off, you will not catch them all.

## Port Scanning

Go to a few companies and ask for a network overview of all Windows servers. You'll be surprised how many companies don't have such a plan. These Servers also offer services. TCP pots are used to connect to these applications. This takes me to another powerful Cmdlet: Test-NetConnection.
Test-NetConnection can perform a ping like Test-Connection. But there is more. It enables you to do additionally tests, such as displaying TCP connection states and routing information. Running in Detailed Mode, it shows the Next Hop Address (usually assigned to a Router Device) and the DNS Server responsible for name resolution.
```
PS C:\> Test-NetConnection -ComputerName sid-500.com -InformationLevel Detailed

ComputerName           : sid-500.com
RemoteAddress          : 2001:aefb::1
NameResolutionResults  : 2001:aefb::1
                         10.0.0.8
                         10.0.0.7
InterfaceAlias         : Ethernet 2
SourceAddress          : 2001:aefb::1
NetRoute (NextHop)     : ::
PingSucceeded          : True
PingReplyDetails (RTT) : 0 ms

PS C:\>
```
With Test-NetConnection it is easy to check the status of TCP ports. Just enter the remote computer name or IP-Address and the port number.
```
PS C:\> Test-NetConnection -ComputerName sid-500.com -Port 443


ComputerName     : sid-500.com
RemoteAddress    : 2001:aefb::1
RemotePort       : 443
InterfaceAlias   : Ethernet 2
SourceAddress    : 2001:aefb::1
TcpTestSucceeded : True

PS C:\>
```
The TcpTestSucceeded returns True, which means that a connection to the destination host with port 443 was successfully established.

You may now think nmap is the best tool for port scanning. Why now using another tool? Sure, Test-NetConnection can't keep up with nmap, but Test-NetConnection is available out-of-the-box on every Windows operating system. And it was never the target to compete with nmap. With this in mind, let's make some improvements on Test-NetConnection.

## Function Test-OpenPort

The downside of Test-NetConnection is that we can't scan multiple computers and multiple ports at once.
This fact gave the idea to write an advanced function to test multiple computers and multiple ports simultaneously. Let’s start with the code. It's a function with 
two parameters target and port. The port parameter is a mandatory parameter. If no computername is specified, localhost will be scanned.

```
function Test-OpenPort {
[CmdletBinding()]
param
(
    [Parameter(Position=0)]
    $Target='localhost', 
    [Parameter(Mandatory=$true, Position=1, Helpmessage = 'Enter Port Numbers. 
    Separate them by comma.')]
    $Port
)

$result=@()
foreach ($i in $Target)
    {
        foreach ($p in $Port)
            
            {
        
                $a=Test-NetConnection -ComputerName $i -Port $p -WarningAction SilentlyContinue
                $result+=New-Object -TypeName PSObject -Property ([ordered]@{
                                'Target'=$a.ComputerName;
                                'RemoteAddress'=$a.RemoteAddress;
                                'Port'=$a.RemotePort;
                                'Status'=$a.tcpTestSucceeded
                                                                   })                             
            }
    }
Write-Output $result
}
```
Running the command with multiple computers and multiple ports shows all port states by computer.
```
PS C:\> Test-OpenPort -Target microsoft.com,10.0.0.7 -Port 443,53,88

Target        RemoteAddress  Port Status
------        -------------  ---- ------
microsoft.com 23.100.122.175  443   True
microsoft.com 23.100.122.175   53  False
microsoft.com 23.100.122.175   88  False
10.0.0.7      10.0.0.7        443   True
10.0.0.7      10.0.0.7         53   True
10.0.0.7      10.0.0.7         88   True

PS C:\>
```
Back to nmap and PowerShell. I always try to avoid 3rd party tools wherever possible. Why? These tools have to be installed. The problem is that they might not be available on other systems and therefore I have to work with onboard resources, even if the software is available, who says I have the rights to install it? Certainly, also my function shown here is not on board by default. But the built-in port scanner called Test Connection.

 I can't stress it enough, working with onboard tools sets you apart from others. This is a wonderful transition to the topic monitoring with onboard resources.

# Monitoring your Network with PowerShell

Unfortunately there are still many environments that do without monitoring their systems. Monitoring the availability of services and applications is a decisive factor for stability. You probably want to monitor your network at regular intervals. I recommend to monitor all your services! You also want to be notified in the event of a server failure. Anyway, I sleep better knowing that everything's all right. In this part I'll put two tools in your hand that enables you to monitor your Windows Servers in terms of availability and security.

## Monitoring the Availability of Domain Controllers

I always assume that all Active Directory Domain Controllers are up and running. This is not always the case with Windows client computers. Based on this fact, we can use ping to monitor all these DCs. The **Try Catch Block** in our function catches all unreachable servers and sends an e-mail message with all important information about these servers. Note, that you have to adapt the Send-MailMessage line to your requirements. You can modify the catch block as you see fit, for example you could write all unavailable DC to a log file or send it as a message as shown in the example above.
```
$dcs=(Get-ADDomainController -Filter *).Name
foreach ($item in $dcs) {
 Try
 {
 Test-Connection $item -Count 1 -ErrorAction Stop | Out-Null
 }
 Catch
 {
 $Site=(Get-ADDomainController $item).Site
 $IP= (Get-ADDomainController $item).IPv4Address
 $date=Get-Date
 Send-MailMessage -From Alert@domain.com -To p.gruenauer@domain.com -SmtpServer EX01 -Subject "Site: $Site | 
 $item is down" -Body "$IP could not be reached at $date.`n`nIf you receive this message again in 15 minutes, $
 item is probably down."
}
}
```
If one of the Domain Controller fails, I will get a message with detailed information about this Domain Controller.

Image

Great! Who would have believed that it is possible to implement a monitoring system in 10 minutes.

By the way, you can easily modify my script to monitor all domain-joined WindowsServers. In order to do that you have to catch them all by using Get-ADComputer.
```
PS C:\> (Get-ADComputer -Filter 'operatingsystem -like "*server*"').Name
AzDC01
AzServer01
AzDC02
```
PS C:\> 

Don't forget to customize the Catch Block too. Since these are not just domain controllers, remember that Get-ADComputer now gets its properties, and this cmdlet has different properties than the Cmdlet Get-ADDomainController.

## Monitoring Windows Firewall

Windows Firewall was first introduced with Windows XP SP2. That was a very long time ago. Unfortunately, in my experience, the host based Windows Firewall is still disabled on many systems. This is partly because administrators have not really dealt with this security feature yet, or because they have simply been deactivated it to see if it is the Windows Firewall´s fault that something is not working. Neither reason is a good one. Meanwhile most attacks come from an ***attacker sitting inside the company network***. The company-wide application firewall cannot help to prevent attacks here. Therefore it is important that the Windows Firewall is enabled on all Windows operating systems. If you are a Windows Server administrator, make sure your systems are well protected, in short keep an eye on it.

***Get-NetFirewallProfile*** is your friend when it comes to show the current state of the Windows Firewall. To check if Windows Firewall is on for the Domain Profile retreive the Enabled Property with Select-Object.
```
PS C:\> Get-NetFirewallProfile -Profile Domain | Select-Object -ExpandProperty Enabled
True
PS C:\>
```
To get the current state of the Windows Firewall from all domain-joined Windows Servers enter the following code and hit enter.
```
$fresult=@()
$servers=(Get-ADComputer -Filter * -Properties Operatingsystem | 
          Where-Object {$_.operatingsystem -like "*server*"}).Name

foreach ($s in $servers) {

        $check=Invoke-Command -ComputerName $s -ScriptBlock {Get-NetFirewallProfile -Profile Domain | 
        Select-Object -ExpandProperty Enabled} -ErrorAction SilentlyContinue

        $fresult+=New-Object -TypeName PSCustomObject -Property ([ordered]@{

        'Server'= $check.PSComputerName
        'FirewallEnabled' = $check.Value

        })
        
        }

Write-Output $fresult
```
This gives you an overview of which servers have activated or deactivated the firewall.
```
Server     FirewallEnabled
------     ---------------
AzDC01     False          
AzServer01 True           
AzDC02     True  
```   
Surely, this can't tell you which firewall rules are enabled and and allows traffic on a specific port, but even that can be found out remotely. For example, if we want to find out if AzDC01 allows inbound https traffic, we can easily do so with Get-NetFirewallRule. Note that you have to provide the computername in the ***CIMSession*** parameter. 
```
PS C:\> Get-NetFirewallRule -CimSession AzServer01 -DisplayName "World Wide Web Services (HTTPS Traffic-In)" | 
Select-Object -Property DisplayName,Enabled,Inbound,Action

DisplayName                                Enabled Inbound Action
-----------                                ------- ------- ------
World Wide Web Services (HTTPS Traffic-In)    True          Allow

PS C:\>
```
# Conclusion

Whether monitoring the network, troubleshooting, obtaining IP settings, the possibilities seem to be endless. For all those who have never worked with the network Cmdlets before and not have created some scripts out of them I hope to have prepared a good introduction to the topic. I also hope that for those who are not new to the subject, I have brought up something new and sparked some ideas how to get more out of PowerShell and Networking. My main concern was to give food for thought. But also some ready-made scripts were included. In that sense, I hope you enjoyed it and have fun with the next chapter!












