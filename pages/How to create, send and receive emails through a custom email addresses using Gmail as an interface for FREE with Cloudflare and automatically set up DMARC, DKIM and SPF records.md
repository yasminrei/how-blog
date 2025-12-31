# How to create, send and receive emails through a custom email addresses using Gmail as an interface for FREE with Cloudflare and automatically set up DMARC, DKIM and SPF records

Prerequisites:
- Domain name already purchased
- Domain name connected to Cloudflare (this is done automatically if you purchase your domain through Cloudflare)
## Setup for receiving emails

1. Go to Cloudflare > Email > Email Routing
2. If you have not yet added any emails, you will need to click "Get Started". If you already have added emails, go to the Routing rules tab of Email Routing and select Create Address
3. Enter the address you would like to create. Ensure action is set to "Send to an email". Then type in the destination Gmail address you would like to forward this email to. Then hit "Create and continue" or "Save".
4. You may be prompted to verify your destination address. Go to your Gmail inbox and click "Verify email address".
5. You will be taken back to the Email Routing dashboard. Click on "Enable Email Routing" which will set up all the MX records. These are your DMARC, DKIM and SPF records.
⚠ Check that you are not using this domain to send any other kind of emails. If you've already set up something like Google Workspace on this domain, it's going to disable all those systems. The only way this email is going to work through this domain, once we press add (MX and TXT)records and enable, is through this email forwarding, so you want to make sure you don't have any kind of emails setup on your domain.
If the records you have already set up are for forwarding, then it's fine. You can leave the MX and TXT records as is.

## Setup for sending emails
1. Go to the Gmail inbox where you have set the destination address to be. 
2. Click on your profile picture and select "Manage your Google Account".
3. Search for 2-step verification under Security. If it is off, select "Turn on 2-Step Verification"
4. Search for App passwords under Security.
5. Enter an App name, make sure it is specific and describes exactly what the password is for so that you are aware in the future.
6. Copy the generated app password.
7. Go back to your Gmail inbox and select Settings > See all settings > Accounts and import
8. In the "Send mail as" row, select "Add another email address"
9. Enter the custom email address you created and keep "Treat as an alias" selected
10. For the SMTP server, type in smtp.gmail.com, the username being your full gmail address for your gmail account, and the password being the app password you just generated and copied. Keep the port as 587 and TLS connected.
11. A confirmation email was sent to your gmail account, so go ahead and click the link to continue with your confirmation in that email.
12. From your Gmail inbox go to Settings > See all settings > accounts and import. Under "Send mail as", select "Reply from the same address to which the message was sent" if you wish to send replies through the same custom email address that received the email.