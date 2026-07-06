# How to setup a GreenCloud VPS and connect to it via hostname with Cloudflare

## Booting the VPS
1. Purchase your VPS from GreenCloud. Mine cost $25 with the following stats:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;IPv4: 1  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;IPv6: /64  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Bandwidth: 4TB  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Port: 10Gbps  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OS: Linux  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Location: Coventry, UK  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Control Panel: Virtfusion  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Backup/Snapshot: 1 Free  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;No refund/Money back on this plan  

2. Go to the GreenCloud control panel.
3. Go to Servers and select Build.
4. Enter a name for your server. I chose ‘My App’.
5. Enter a Hostname for your server. I chose ‘my-app.mydomain.com’.
6. Select the relevant timezone.
7. Select the operating system. I chose Debian 13 (Trixie) Minimal.
8. Choose the Swap Space amount. I chose 2 GB.
9. Under SSH Keys, select ‘Add Key’.
10. Go to your Terminal app. With PowerShell, type in ‘Get-Content $HOME\.ssh\id_ed25519.pub’. If the public ssh key of your personal computer is named something else, change the file path accordingly.
11. Copy the response from the terminal. It should start with something like ‘ssh-ed25519’ and end with something like ‘your@email.com’. Don’t copy in any whitespaces at the start or the end of the string.
12. Go back to the GreenCloud page and paste in the key you just copied into the Public Key box. 
13. Enter a name for your key.
14. Press ‘Install’.

## Linking the hostname to the IP in Cloudflare
1. Go to Cloudflare and select the domain that you used as part of your hostname when installing your server.
2. Go to DNS > Records
3. Press ‘Add record’.
4. Keep the record type as ‘A’ and enter the subdomain you used as part of your hostname into the ‘Name’ field.
5. Go back to the GreenCloud dashboard and go to the Servers tab. Scroll down until you see Network and copy the IP address in the top right of the Network section.
6. Go back to Cloudflare the IP address into the IPV4 address box.
7. Hit Save.

## SSH into your server via hostname
1. Go to the terminal app on your computer. I used PowerShell.
2. Replace the word ‘hostname’ from this command with your actual hostname that you entered when booting your GreenCloud VPS and copy the whole command: ssh root@hostname -p 22
3. Paste the command into your terminal.
4. If you see a ‘Last login’ message with root@subdomain on the left hand side, you are in
