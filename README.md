# ADLS-SFTP-AZFW
ADLS SFTP Endpoint behind Azure Firewall

For those folks who work in highly regulated industries(or just have tight security requirements to meet) and want to use the new SFTP endpoint for blob -- the solution below outlines the method you can use to put ADLS + SFTP behind an Azure Firewall secured in your VNET.

A simple illustration of what we're building:

<img width="726" alt="image" src="https://user-images.githubusercontent.com/15015304/218517798-408d688e-74b2-4815-a29e-b4e1b2930e28.png">

Below are the services we'll use for this sample:

Azure Firewall (I used the Basic SKU in my lab to keep costs down, but any SKU will work)<br>
An additional public IP to act as the SFTP VIP on my firewall<br>
Two VNETs (This solution uses a basic hub/spoke network example)<br>
Blob Storage Account (+ADLS/HNS & SFTP Enabled)<br>
Private Endpoint (for Blob)<br>
Private DNS Zone (to faciliate the private endpoint name resolution)

Go Ahead and create(or use use existing) your VNETs for this lab. Make sure when you create each (Hub and Spoke) that you ensure you don't use overlapping IP   addresses, as we'll need to use VNET peering to connect the two. Once both are created, peer the two VNETs together in preparation for our following steps.

For each VNET, we'll need the following subnets defined:

HUB-VNET:<br>
AzureFirewallSubnet<br>
AzureFirewallManagementSubnet

SPOKE-VNET:<br>
PepSubnet (for our private endpoint)

Once your subnets are in place, go ahead and deploy the Azure Firewall of your choice. Again, I used the basic SKU to keep costs down, but any will work. After your firewall is created, let's create an additional public IP that we'll use specifically for SFTP. When you create this public IP, go ahead and give it a DNS name label so we can address the SFTP endpoint over a public name of our choice.

<img width="178" alt="image" src="https://user-images.githubusercontent.com/15015304/218505587-53851caf-2fc3-4f00-a60e-9d4f98e0a09e.png">

Then, go into your firewall and add the public IP under 'Public IP Configuration' -- we'll use this in our DNAT rule later.

<img width="402" alt="image" src="https://user-images.githubusercontent.com/15015304/218505883-993c1433-9a6b-44e2-89ca-caab9c3ffe04.png">

Now, create your blob storage account that we'll use for SFTP. During the creation flow, ensure it's marked as ADLS as well as enabled for SFTP. Once the storage account is created, go ahead and create a container that we'll use as an SFTP target (mine is called landingzone). Afterwards, navigate to the 'SFTP' section on the storage account blade and let's create a local user for SFTP. Also, make sure you define the users 'home landing directory' when you do this -- it should be in the format %containerName%/folder. (my test user for example would be landingzone/testuser)

<img width="471" alt="image" src="https://user-images.githubusercontent.com/15015304/218506768-3eaacb8e-1260-4f8e-8464-a86a43a65035.png">

After the user creation is complete, your SSH password(assuming you chose that authentication method) will appear -- take note of this for later.

<img width="258" alt="image" src="https://user-images.githubusercontent.com/15015304/218507053-f46c54ab-9abe-49ba-91cd-bf2ca19f755b.png">

Once this is all done, let's create a private endpoint for the Blob w/SFTP. You can do this from the 'Networking' section of the storage account blade. If you don't have a private DNS zone in place for 'privatelink.blob.core.windows.net', the UI in the Portal will create this for you. The private endpoint should be created in the 'PepSubnet' subnet we created during the VNET creation process earlier.

Now, let's create the DNAT rule on the Azure Firewall to do the necessary NAT from the public IP to the back-end SFTP storage target. In my example below, the rule breakdown is as follows:

Source type: IP Address<br>
Source: <my home pulbic IP address> (I just grabbed my source address using 'Whats my IP' in a Google search bar, but any method you want to use is fine) My goal was to ensure I was only allowing select source IPs (from the Internet) hit my SFTP endpoint.<br>
Protocol: TCP<br>
Destination Ports: 22<br>
Destination Type: IP Address<br>
Destination: 20.81.84.171 (This is the public IP I created on my firewall specifically for SFTP)<br>
Translated Type: FQDN<br>
Translated address or FQDN: rnusftpendpoint.blob.core.windows.net (this is the public name of my blob storage account. the DNS recursion process from within the firewall will ultimately translate this to rnusftpendpoint.privatelink.blob.core.windows.net since we have a private endpoint configured)<br>
Translated Port: 22

<img width="541" alt="image" src="https://user-images.githubusercontent.com/15015304/218507690-ad48c238-9520-4efa-83c1-3840c5ef1a05.png">
<img width="296" alt="image" src="https://user-images.githubusercontent.com/15015304/218507734-5d937c46-9f0d-4323-9e07-89b13d2ebc90.png">


Now, we have all the required services in place and configured to support exteranl SFTP. The next thing left to do is test from our SFTP client. This is a pretty straightforward step if you've used SFTP before -- you'll just need to supply it with your remote hostname and username. Below is the format you'll need to use when doing this:
  
Remote host: rnusftp.eastus.cloudapp.azure.com (This is the DNS name we appended to our SFTP public IP on the firewall. You can customize this record if you wish to support a vanity name, but I just used the default suffix)<br>
Username: rnusftpendpoint.testuser (this format is important -- it should be constructed using %storageAccountName%.%LocalFtpUserName%
  

That's it -- you should now be able to connect to your SFTP endpoint (from the Internet) and have the traffic routed through and inspected by your Azure Firewall. This will allow you to control flows as well as use your firewall for a centralized traffic inspection point.

