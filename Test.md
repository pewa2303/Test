# Prerequisites

It is assumed the reader has an intermediate knowledge of PowerShell and Network Technologies. 

The examples in this chapter require an Active Directory domain environment with at least one Windows Server 2016 domain controller and one Windows Server 2016 member server. No prerequisites for forest and domain mode. Make sure Windows remote management (WinRM) is enabled on all computers.

## Introduction

The more I play with it, the more I like it: PowerShell and the networking Cmdlets. Everybody knows ping. Everyone knows arp. But only very few have dealt with advanced network tasks in PowerShell. I want to change that here and will give the reader many useful tips to demonstrate that the object-oriented PowerShell has a lot to offer when it comes to doing network tasks, monitoring, automation and troubleshooting. Why do I actually think it’s worth talking about PowerShell and Network Technology?
In over 20 years of experience in IT, I can't tell how many times I've heard those words: "Implementing a monitoring solution will take a lot of time and cost a lot. We can't afford that. Now we have no choice but to hope that everything works fine and no errors occur on our production servers." This means that in the event of errors and failures, there is a greatly delayed reaction and thus a longer downtime. But these people are right, implementing a centralized monitoring solution will mean much effort, but does it really have to be an enterprise solution? Can't we build a small monitoring solution in a few simple steps? In any case, anything is better than doing nothing. That`s what I believe.
The same goes to network security. "We have servers with a graphical user interface as we are used to. The large number of Windows updates are a little annoying, but that's just the way it is with Windows. We're used to it, it's always been that way." No, not anymore. With the introduction of Windows Server Core and Nano Server, the Windows world has changed. Server Core and Nano Server require fewer updates, less storage space, and are more secure than any other Windows operating system so far due to the smaller number of features installed. 
All these examples bring us to our topic for this chapter: PowerShell and network technology. With only a small set of Cmdlets in hand and some scripting know how we are able to do wonderful things at zero cost.
Let’s dive in.
