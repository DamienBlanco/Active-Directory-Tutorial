# Active-Directory-Tutorial

Setup

Step 0: Download a copy (ISO) of Windows Server 2019 https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019  
Then create a copy of your preferred Pro Windows version (10 in my case) using the media creation tool https://www.microsoft.com/en-us/software-download/windows10

Step 1: https://i.imgur.com/wyAT3WV.png In the Hyper-V Manager create a new switch for the internal network using the Virtual Switch Manager. When creating the virtual machine attach two switches. The default will connect to the internet, the internal switch will provide DHCP to any other clients connected to the server. If you don't want to use the default switch, you can create a new virtual switch and configure NAT through PowerShell.

(Note: At the time of writing there is an issue with W10 Hyper-V sharing the physical network card - both connections (on the host and on the VM) will become unusable. There is a workaround doing the following: create a second internal virtual switch, go to your host machine's internet adapter, enable sharing in the properties and select this second internal passthrough switch, use this passthrough switch as a substitute for an external switch)

Step 2: https://i.imgur.com/FSt5cGs.png The external network adaptor will automatically obtain an IP. The second (internal) network adaptor will be configured as such using a private IP, and the loopback address for DNS.

Step 3: https://i.imgur.com/kINSbHv.png In the server manager, select "Add roles or features" and select what you want to install. I will be installing Active Directory, Remote Access, DHCP Server in "Server Roles", routing in "Role Services" . 

Step 4: Promote the server to a Domain Controller (yellow flag at the top). Add a new forest, and hit next a few times. When you sign back in, be sure to sign in using the MYDOMAIN\ADMINISTRATOR (or whatever your username is)

Step 5: In the AD Users & Computers page you can add OUs and users. Create a new user for yourself and give it admin privileges (add it to the Domain Admin group). Log in and use this user from now on.

Routing

Step 6: In the Server Manager open tools, and then "Routing and Remote Access". "Configure and enable routing & Remote access", install NAT using the network controller that connects to the internet. 

DHCP

Step 7: Open the DHCP control panel from the Server Manager. Configure the IPv4 address range as 172.16.0.100 - 172.16.0.200 / 24. Lease duration can stay at default or reduced to a few days. Configure the DHCP options, 172.16.0.1 (be sure to add), mydomain.com.  https://i.imgur.com/bexpa1b.png

Step 8: Open IPv4 Server options, add 003 Router, input the IP of the DHCP host, and restart the domain controller

	• Step 8.1: Navigate to DNS inside server manager's tools. Open your DC's properties, uncheck the IPv6 address if you are not using it. Under "Forwarders", make sure you have another DNS server set, either what your ISP recommends, or simply using Google's 8.8.8.8 otherwise. 

Step 9: Create a client VM. Attach the same internal switch for DHCP from the domain controller. Install W10 Pro. 

https://i.imgur.com/lJ8hNq8.png Success! We can confirm DHCP connectivity and DNS resolution!

Step 10:  Navigate to system -> Rename this PC (advanced) -> Change. Rename the Computer Name (Client1 in my case) and set it as a member of mydomain.com. Input one of the accounts you configured in step 5 to successfully join the domain (by default a user can join 10 of their PCs to a domain without Admin approval - governed by 
ms-DS-MachineAccountQuota attribute)

https://i.imgur.com/Wxmzpbx.png 

Over on our domain controller, inside the DHCP manager, we can see inside the address leases folder that our client1 pc has successfully leased an IP address https://i.imgur.com/lkeEySQ.png

Shared Folders

Step 11: Create a new OU for the group we're going to add - in this case it will be "Springfield". Create two more OU: one for security groups, and one for the families. Inside the Families OU we can create separate family folders where we will put each individual family member (e.g all the Simpson's go into the Simpsons folder). 

Step 12: Navigate to Files and Storage Services in the Server Manager. Click on the Shares tab and then create a new Share, SMB Share - Quick. In the default drive, create a location you want to use for the Share Location ("Springfield") and select it in the custom path at the bottom. In "Other Settings" enable access-based enumeration. In the Permissions section,  select Customize Permissions and disable inheritance. Remove the default user group, make sure the Springfield group is added. Ensure that your "Springfield" group (with all residents) can modify, read, list, write, and execute. 

All Springfield residents will have access to this shared folder. Inside the shared folder you can create separate folders for each family ("Flanders church Hymns", "Nuclear Power Plant Blueprints", etc.). In each separate folder you can specify which group has access using the following method: Right click the location in file explorer, under sharing select advanced sharing, share the folder, permissions, remove "Everyone", and replace with the group you want to give access, modifying change or read as wanted. Finally, under security, edit, add the relevant group, give them relevant permissions (usually modify through write).

(Note: There is a common issue with Hyper-V enhanced session that results in clients being treated as remote logins. You can disable enhanced session or add Springfield user group into the "Remote Desktop Users" security group in the Builtin domain folder)

https://i.imgur.com/O8SI1LE.png

Here we are logged in as Homer Simpson. Homer can access Simpson Family Photos, Moe's Bar Chatroom, and Nuclear Power Plant Blueprints, but he can't access Flanders' folder or the Elementary School Assignments folder as he lacks those permissions. 

Group Policy

Step 13: Open the "Group Policy Management Editor". From here we will create a few baseline GPOs for our users. Give descriptive names and / or use comments.

	•  Network Drive Mapping - link a new GPO to Springfield. Edit > User Configuration > Preferences > Window Settings > Drive Maps > New Mapped Drive. Add the share link to the Location and Show this drive.
	
	• USB Permissions - For this GPO I want to limit the access of USB devices that children have. I created an OU with a single security group called "Minors", and placed every underage user inside of it. Create a new GPO and map it to this OU.  
	
	Edit > Computer Configuration > Policies > Administrative Templates > System > Removable Storage Access. Change the following settings to enabled: "Removable disks: Deny read access, Removable disks: Deny write access". Minors will now be restricted from using USB media, however, they can still utilize other media such as CDs or Floppy disks if necessary for school.   
	
And with this we are done our basic Active Directory lab! We've successfully configured DNS and DHCP, created a new forest for our fleet of users, organized them using security groups, created shared locations that only certain groups can see, and applied some baseline restrictive GPOs according to the needs of our group.
