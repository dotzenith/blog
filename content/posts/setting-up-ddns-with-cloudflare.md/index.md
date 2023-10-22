---
title: How to set up Dynamic DNS for Cloudflare
date: 2023-10-21T19:07:47-04:00
description: Because paying for a static IP is expensive
---

# Intro 

As someone too poor to pay for a static IP, I struggled a little when I wanted to set up a VPN tunnel into my home network so I could access my self-hosted applications. Fortunately, getting DDNS working is actually quite easy and it only takes a few minutes. 

I'll be using [inadyn](https://github.com/troglobit/inadyn) for this article since it's not nearly as buggy as [ddclient](https://github.com/ddclient/ddclient). As the title suggests, I'll only cover the configuration for Cloudflare but inadyn supports a lot of different [providers](https://github.com/troglobit/inadyn#supported-providers) with [examples](https://github.com/troglobit/inadyn#example) for all of them as well. This guide also assumes you're using linux.

# Requirements

- [inadyn](https://github.com/troglobit/inadyn) `v2.12.0` or above
- A Cloudflare API Token with `Zone:Zone:Read` and `Zone:DNS:Write` permissions

# Installing [inadyn](https://github.com/troglobit/inadyn)

Navigate over to [install instructions](https://github.com/troglobit/inadyn#build--install) and follow the section relevant for your distro.

I found that the [debian package](https://packages.debian.org/sid/inadyn) was still on `v2.11.0-1` so I decided to install from source.

The steps are as follows: 

```sh
# Download the source
curl -OL https://github.com/troglobit/inadyn/releases/download/v2.12.0/inadyn-2.12.0.tar.gz

# Extract and cd into extracted folder
tar xf inadyn-2.12.0.tar.gz && cd inadyn-2.12.0

# Install dependencies
sudo apt install gnutls-dev libconfuse-dev
sudo apt install build-essential pkg-config

# Configure and build
./configure --sysconfdir=/etc --localstatedir=/var
make -j5
sudo make install-strip
```

If the above commands succeed without any errors, you're good to go. If not, please refer to the [build instruction](https://github.com/troglobit/inadyn#building-from-source) again.

# Getting the Cloudflare API key

1. Login in to your Cloudflare account and on `My Profile` at the top right
2. Click on the `API Tokens` tab in the sidebar
3. Click `Create Token` and then use the `Edit zone DNS` template
4. In the permissions section, add `Zone:Zone:Read` and `Zone:DNS:Edit`
5. For Zone Resources, select `Include -> All Zones`
6. You can leave the Client IP Address Filtering section as-is
7. Select how long you'd like the token to be active for in the TTL section
8. Click `Continue to summary`
9. Verify everything and click `Create Token`
10. Copy the token and keep it saved, you will only see this once

# Set up inadyn

Now that we have all of the pre-requisites taken care of, we're finally ready to set up the Dynamic DNS service

Open `/etc/inadyn.conf` in the text-editor of your choice and paste the following:

```
period = 300 # How often the IP is checked. The value denotes seconds

provider cloudflare.com {
    username = yourdomain.com
    password = your-api-token
    hostname = hostname.yourdomain.com
    ttl = 1
    proxied = false # optional.
}
```
> Note: Make sure you have a corresponding A record for the `hostname` in your Cloudflare dashboard

Once the config file is taken care, we can enable and start the service by running the following:

```
sudo systemctl enable inadyn.service
sudo systemctl start  inadyn.service
```

And you can check the status running:

```
sudo systemctl status inadyn.service
```

If all goes well, inadyn should update the specified hostname with the IP of your host machine

> Note: inadyn will only update the IP if there's a change, but you can force an update by specifying the `forced-update = 300 # The value denotes seconds` config option

# Conclusion

I hope this serves as an easy and straightforward guide to setting up DDNS with Cloudflare. Next up, How to set up a wireguard VPN tunnel into your home network :)
