Installation Of Active Directory On Windows Server 2019 Core
---
In this tutorial we will setup active directory in Windows Server 2019 and we will also setup a Windows 10 client to work as a workstation, by installing RSAT for windows 10.





After the initial Install of server core and reboot, the first screen that comes up, user's password must be changed before signing in. Just hit enter on ok.

![ok](/pic/2.png)

Then enter a password.

![pass](/pic/3.png)

 Then enter `` start powershell ``

![d](/pic/4.png)

In Virtual machine select Install VMWare tools. In poweshell type `` cd D: ``
and then ``ls `` which will allow you to see what is in the D directory, we want setup64.exe so type `` .\setup64.exe ``

![power](/pic/5.png)

sometimes VMWare tools will start behind the powershell window, just minimize the window and complete the setup.

If you ever accidentally close the command prompt window, press Ctrl+Alt+Delete, open the Task Manager -> File -> Run -> and run cmd.exe (or PowerShell.exe).

I have a lot of scripts that I use because I cannot remember all the powershell commands,. I sometimes go to help in powershell to find these commands , however with a script that I have handy I will just copy and paste. Now I work a lot with VMWare and I find sometimes when Im using windows core I'm not able to copy and paste, make sure you select Use Ctrl+Shift+CV as Copy Paste in powershell properties

![copy](/pic/7.png)

Now if that doesn't work I will use SSH in my terminal to connect to windows core. On my mac I will just type in `` ssh administrator@and the IP address of the server ``
Now before we can SSH into our server we need to do a few things. We need to Set the IP Address. Type in : `` New-netIPAddress -IPAddress 192.168.95.20 -PrefixLength 24 -DefaultGateway 192.168.95.2 -InterfaceAlias Ethernet0 ``

Now the next thing we type in is to set our DNS ServerAddresses

Type in: `` Set-DNSClientServerAddress -ServerAddresses 9.9.9.9 -InterfaceAlias Ethernet0 ``
We will change this later after we install an OpenSSH server

Let's disable IPv6: Type this : `` Disable-NetAdapterBinding -Name 'Ethernet0' -ComponentID 'ms_tcpip6' ``

Now let's make sure of the change, type in: `` ipconfig /all ``

![ip/all](/pic/8.png)

Next thing is to set up open SSH. So let's run this in powershell to see what is installed already.

`` Get-WindowsCapability -Online | ? Name -like 'OpenSSH*' ``

![ssh](/pic/9.png)

So OpenSSH Server is not installked, lets install by typing in: ``  Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0 ``

![result](/pic/10.png)

# Start the sshd service
`` Start-Service sshd ``

# Set the service to automatic start
`` Set-Service sshd -StartupType Automatic ``

Now in your terminal type:

`` ssh administrator@192.168.95.20 ``


![term](/pic/11.png)


Select yes, and then enter the password of your server

So now we are connected to our server, I also changed the backgoud and text color to look like a powershell terminal    

![term](/pic/13.png)

Now in your terminal enter:

`` powershell.exe ``

You can see that in the terminal it starts with PS (Powershell)

![PS](/pic/14.png)

So let's get info about our Windows Server version:

`` Get-ComputerInfo | select WindowsProductName, WindowsVersion, OsHardwareAbstractionLayer ``   

`` $PSVersionTable ``


![version](/pic/15.png)


To allow remote access to Server Core via RDP:
`` cscript C:\Windows\System32\Scregedit.wsf /ar 0 ``

![result](/pic/16.png)




Set TimeZone:

  `` Tzutil.exe /s "Central Standard Time" ``

or this:

`` Set-TimeZone "Central Standard Time” ``       


Let's change our DNS Server addresses to 127.0.0.1 and 9.9.9.9

 `` Set-DNSClientServerAddress -ServerAddresses ("127.0.0.1,9.9.9.9") -InterfaceAlias Ethernet0 ``

 ![dns](/pic/17.png)

Now lets Rename ourComputer

`` Rename-Computer -NewName DC1 ``

So because your logged in on your mac terminal, you will have to exit and restart through powershell on your VM. Otherwise you will get an error.

  `` Restart-computer ``






Install AD & DNS
--

Now We will install ADDS Role and Mgt Tools

`` Install-WindowsFeature AD-Domain-Services -IncludeManagementTools ``


![ad](/pic/18.png)


Now we Install-ADDSForest:



`` Install-ADDSForest -DomainName ‘cloudsoc.local’ `
-ForestMode WinThreshold `
-DomainMode WinThreshold `
-DomainNetbiosName CLOUDSOC `
-SafeModeAdministratorPassword (ConvertTo-SecureString -AsPlainText “green@123” -Force) `
-InstallDNS ``

This will ask you for the Domain Name and SafeMode password. Make sure you write this down in a safe place. It will be very useful in case of disaster recovery.

You can select Y or A as an answer to the question.

While installation is going on you will see some warning.

![warn](/pic/19.png)


Your server will automatically reboot. And you will have to close your mac terminal and sign back in. Notice your VM terminal will be applying settings, and we will not be able to login with our mac terminal until its finish.

So now let's check out our DNS server zone.

`` Get-DnsServerZone ``

![zone](/pic/20.png)

Everything looks good , now lets create reverse lookup zone.


`` Add-DnsServerPrimaryZone -NetworkID 192.168.95.0/24 -ZoneFile “95.168.192.in-addr.arpa.dns" -DynamicUpdate None -PassThru ``

![reverse](/pic/21.png)



Now lets add a pointer record in our reverse lookup zone.

`` Add-DNSServerResourceRecordPTR -ZoneName 95.168.192.in-addr.arpa -Name 20 -PTRDomainName dc1.cloudsoc.local ``




Now in our mac terminal let's do a nslookup and ping cloudsoc.local

`` nslookup cloudsoc.local ``

`` ping cloudsoc.local ``

![ping](/pic/22.png)





Now in active directory users and computers, lets add an OrganizationalUnit and name it Windows Admin.

`` New-ADOrganizationalUnit -Name "Windows Admin" ``




Now lets add a user to Windows Admin.

`` New-ADUser -Name "Jack Robinson" -GivenName "Jack" -Surname "Robinson" -SamAccountName "J.Robinson" -UserPrincipalName "J.Robinson@cloudsoc.local" -Path "OU=Windows Admin,DC=cloudsoc,DC=local" -AccountPassword(Read-Host -AsSecureString "Input Password") -Enabled $true ``

It will ask for a password , just enter what you want.

Now lets get that new users info.




`` Get-ADUser J.Robinson ``


Setup a Windows 10 client to work as a workstation, by installing RSAT for windows 10.
--

I'm using a windows 10 Enterprise for my workstation, I could have also used Windows Pro, the windows home version will not work.

So in windows 10 open powershell admin and we are going to change the name of our computer to CloudSOC1.

`` Rename-Computer -NewName CloudSOC1 ``

 And

 `` Restart-computer ``


Now right click on the Internet access icon in your taskbar and select Open Network and Internet settings. Double click on ethernet0, select properties, and then double clicl on Internet Protocol Version 4, and select Use the following DNS server and type in 192.168.95.20.

![dns](/pic/28.png)






Now click on file explorer in your taskbar, and right click on this pc and select properties, this will open up Control Panel System and Security window, click on Change settings for Computer name , then click on change

 ![change](/pic/24.png)

 select Domain and type in cloudsoc.local

 ![type](/pic/25.png)

 In the next window type in administrator and enter your password.

 ![pass](/pic/26.png)

 ![welcom](/pic/27.png)

 Then you must restarter your computer.

 Now this time when you go to log in, select other user and type in:
`` administrator@cloudsoc.local ``
and then enter your password.

I have already logged in as administrator as you can see here:

![login](/pic/29.png)


Now in your browser let's download [RSAT for windows 10](https://www.microsoft.com/en-us/download/details.aspx?id=45520)


Scroll down and Select download and preferred language.

Now when you come to chose download you want, you will have know the version of your windows 10
Mine is Version 1809, so I will select WindowsTH-RSAT_WS_1803-x64.msu

![rsat](/pic/31.png)


 After you download RSAT just install it from which ever folder you saved it to.

 When it finished you will see it in your start menu, just pin it top your taskbar.

 Open server manager and selct Add other servers to manage.

 ![mamage](/pic/32.png)


 Click find now.

 ![find](/pic/33.png)

 Select DC1 and then OK.

 Give it a minute to find the server

![servers](/pic/35.png)

Now in server manager select tools and active directory users and computers.

![users](/pic/36.png)

You can see the organizational unit we created with the user Jack Robinson.

![jack](/pic/37.png)

And when you click on the computer folder you will see the cloudsoc1 computer.

![comp](/pic/38.png)

now go back to tools and select DNS, you may have to enter the computer, which is DC1.

![dns](/pic/39.png)

Go ahead and try out other tools.

That all for this tutorial.
