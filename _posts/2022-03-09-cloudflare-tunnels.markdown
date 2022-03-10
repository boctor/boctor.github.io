---
title:  "Replacing ngrok with Cloudflare Tunnels"
date:   2021-03-09 19:29:00 -0800
categories: cloudflare
comments: true
---
If you've ever used [ngrok](https://ngrok.com) or [smee.io](https://smee.io) you're familiar with the need to have code running locally on your machine but that's triggered by a publicly accessible URL.

It's a great way to test your code before it's deployed. I've often used it for debugging node code that is triggered by another service such as a Github or Slack webhook.

Ngrok used to be open soure but now the free plan only supports creating a new randomly generated URL every time the tunnel starts. When I used ngrok I'd dread when my machine restarted since the tunnels got restarted and I'd have to reconfigure services like Github or Slack with the new URL.

I've switched to Cloudflare Tunnels which support this workflow for free. 

It uses subdomains on your custom domain that's managed by Cloudflare so assuming your domain is mydomain.com, you end up with urls like wedev.mydomain.com. 

Best of all, the urls never change!

Here's a quick guide outlining how to set it up and use it.

## Before you start
To get started you'll need some prerequsites:
* Have or [create](https://dash.cloudflare.com/sign-up) a Cloudflare account
* [Add a website to Cloudflare](https://support.cloudflare.com/hc/en-us/articles/201720164-Creating-a-Cloudflare-account-and-adding-a-website)
* [Change your domain nameservers to Cloudflare](https://support.cloudflare.com/hc/en-us/articles/205195708)
* [Setup Cloudflare Zero Trust](https://dash.teams.cloudflare.com/) You can choose the free option, but you still will be required to provide a credit card.
* Install `cloudeflared` which is the [Cloudflare Tunnel client](https://github.com/cloudflare/cloudflared). On MacOS you can use Homebrew:
  * `brew install cloudflare/cloudflare/cloudflared`
  * (It's like saying Beetlejuice three times but with cloudflare)

## Setup

### Login
Run `cloudflared login` which will open the cloudflare website and prompt you to choose a domain.

After you select a domain, a certificate used to authenticate `cloudflared` will be saved in `~/.cloudflared/cert.pm`

### Create a Tunnel
Replace `new-website` with a name that makes sense for you:

`cloudflared tunnel create new-website`

This will generate a json file with credentials for this tunnel in: `~/.cloudflared/<Tunnel ID>.json`

It will also print out the new Tunnel ID: `Created tunnel new-website with id <Tunnel ID>`

### Configure the tunnel for your project
  
In the root directory of the project where the code you want to run when the public url is visited, create a `.cloudflared` directory.

  In this new `.cloudflared` directory copy the `~/.cloudflared/<tunnel ID>.json` created in the previous step and rename it `credentials.json`

In this `.cloudflared` directory also create a `config.yml` file with the following:
```
tunnel: <Tunnel ID>
credentials-file: .cloudflared/credentials.json
noTLSVerify: true
url: localhost:3000
```
You'll replace
1. `<Tunnel ID>` with the ID of your tunnel.
2. `3000` with the port your code runs on locally.

### Map Tunnel to subdomain
Finally we'll create a new subdomain and map it to this tunnel.

Here you'll replace:
1. `new-website` with the name of your tunnel
2. `webdev` in `webdev.mydomain.com` with the subdomain you want.
3. `mydomain.com` in `webdev.mydomain.com` with your domain.

`cloudflared tunnel route dns new-website webdev.mydomain.com`

You should get a message that a CNAME was added and the tunnel was routed to it:

`Added CNAME webdev.mydomain.com which will route to this tunnel tunnelID=<Tunnel ID>`

## Start Debugging
Now that everything is setup, here are the steps you will repeat whenever you want to start debugging.

### Run your code
Start your code on a port on localhost

### Run your tunnel
In the directory of your project run the following:
`cloudflared tunnel --config .cloudflared/config.yml run`

### Trigger public URL
Visit or trigger a visit to your custom subdomain 