---
title: How to set up Dynamic DNS with Cloudflare
date: 2023-10-21T19:07:47-04:00
description: Because paying for a static IP is expensive
og_size: "30"
---

# Intro 

As someone too poor to pay for a static IP, I struggled a little when I wanted to set up a VPN tunnel into my home network so I could access my self-hosted applications. Fortunately, getting DDNS working is actually quite easy and it only takes a few minutes. 

I'll be using [ZenDNS](https://github.com/dotzenith/ZenDNS) for this article because I wrote it myself to
be simple and easy to use.
As the title suggests, I'll only cover the configuration for Cloudflare but ZenDNS also has support for other [providers](https://github.com/dotzenith/ZenDNS?tab=readme-ov-file#-configuration).

This guide assumes you're using Linux, but you should be able follow on MacOS and Windows as well.

# Requirements

- Working knowledge of the commandline
- [ZenDNS](https://github.com/dotzenith/ZenDNS)
- A Cloudflare API Token with `Zone:Zone:Read` and `Zone:DNS:Write` permissions

# Installing ZenDNS

Navigate over to [install instructions](https://github.com/dotzenith/ZenDNS?tab=readme-ov-file#-installation) and choose the installation option that works best for you.

For me, it was as simple as:
```sh
# Download the source
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/dotzenith/zendns/releases/latest/download/zendns-installer.sh | sh
```

# Getting the Cloudflare API key

1. Login in to your Cloudflare account and on `My Profile` at the top right
2. Click on the `API Tokens` tab in the sidebar
3. Click `Create Token` and then use the `Edit zone DNS` template
4. In the permissions section, add `Zone:Zone:Read` and `Zone:DNS:Edit`
5. You can leave the Client IP Address Filtering section as-is
6. Select how long you'd like the token to be active for in the TTL section
7. Click `Continue to summary`
8. Verify everything and click `Create Token`
9. Copy the token and keep it saved, you will only see this once

# Set up ZenDNS

Now that we have the pre-requisites taken care of, we're finally ready to go

Open `~/.config/zendns/config.json` in the text-editor of your choice and paste the following:
> Note: this file can be anywhere

```json
{
    "providers": [
        {
            "type": "cloudflare",
            "key": "your-api-key",
            "zone": "your-domain.com",
            "hostname": "your-hostname",
            "ttl": 1,
            "proxied": false
        }
    ]
}
```
> Note: Make sure you have a corresponding A record for the `hostname` in your Cloudflare dashboard

Once the config file is taken care of, we can run ZenDNS like so:
```
zendns --config ~/.config/zendns/config.json
```

# Update DNS entries automatically

We very obviously don't want to manually run ZenDNS every time our IP changes, so we can automate it using [cron](https://en.wikipedia.org/wiki/Cron)

We can use cron by running `crontab -e` and then adding this entry:
```bash
*/5 * * * * /full/path/to/zendns --config ~/.config/zendns/config.json --log /var/log/zendns.log
```
> Note: You might need to pass in the full path to ZenDNS because it might no be in the $PATH for cron

This will run ZenDNS every 5 minutes and update the entries if they have changed

> Note: ZenDNS will cache your IP and only make changes if they have changed since last usage, you can override this by passing the `--force` flag

# Conclusion

I hope this serves as an easy and straightforward guide to setting up DDNS using Cloudflare. Next up, we'll set up a wireguard tunnel to easily access our homelab services when we're away.
