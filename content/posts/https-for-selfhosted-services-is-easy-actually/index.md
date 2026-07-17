---
title: "HTTPS for selfhosted services is easy actually"
date: 2026-05-28T20:09:47-04:00
description: Annoying errors and hard to remember ip:port combinations begone!
ogSize: 30
---

# The situation at hand

You've recently fallen into the selfhosting rabbit hole. You've set up a linux server on an old laptop, you've installed docker,
and you've even figured out how to stream **public domain** movies on your [jellyfin](https://jellyfin.org/) instance.

Despite all of that, you have to go to `http://192.168.70.72:8096` every time you want to watch a movie.
A new service you spun up yesterday serves it's own TLS cert, and now you have to say "yes, yes I understand" to every TLS error you see.
You tell yourself this is fine, this is just the cost of hosting your own stuff.

You vaguely know that you can use [certbot](https://certbot.eff.org/) for TLS certificates, but wait doesn't that require opening up port 80?
That's too risky for your home network so you decide to just live like this.

# Better things are possible

You don't have to keep living like the above situation. If you already own a domain, you are not limited to the aforementioned
[HTTP-01 challenge](https://letsencrypt.org/docs/challenge-types/#http-01-challenge) that requires you to open up port 80.

You can instead make use of the [DNS-01 challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge), which just requires
you to have an API key for your registrar/DNS provider, no need to open up ports to the internet.

I use [cloudflare](https://www.cloudflare.com/) as my registrar so the rest of this post will use that for the specific examples.
Don't stress if you don't use cloudflare, the general process should be similar for other registrars as well.

# The solutions at hand

The basic idea here is to use a [reverse proxy](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/) and configure it
to fetch wildcard TLS certificates for your domain using the DNS-01 challenge mentioned earlier.

Given that this post is aimed at new selfhosting enjoyers, my recommendation is to use [Nginx Proxy Manager](https://nginxproxymanager.com/).
It's worth noting that options like [caddy](https://caddyserver.com/) and [traefik](https://github.com/traefik/traefik) exist as well, but I'll leave those as an exercise for the reader.

I will also assume that you have [docker](https://docs.docker.com/engine/install/) installed already.

# How to

Enough worldbuilding, here's how to actually set up TLS for your selfhosted services.
The rest of the How To section will assume that your server's internal IP is something like `192.168.70.72`.
You will of course need to replace that with your own server's internal IP address.

### Configuring Cloudflare

Before we do anything with Nginx Proxy Manager, let's take care of everything we need to do on the cloudflare dashboard.

#### Setting up DNS Records

In the cloudflare dashboard, navigate to the DNS settings tab for your domain, and add the following DNS records:

1. A new `A` record with the following fields:
    ```
    Type: A
    Name: @
    IPv4 address: 192.168.70.72
    Proxy status: DNS only
    TTL: Auto
    ```

2. A new `CNAME` record with the following fields:
    ```
    Type: CNAME
    Name: *
    Target: your-domain.com
    Proxy status: DNS only
    TTL: Auto
    ```

#### Setting up an API Key

I've largely taken these steps from my post on [setting up dynamic DNS](https://blog.danshu.co/posts/setting-up-ddns-with-cloudflare/)
but the steps are the same so that should be fine.

1. Log in to your Cloudflare account and click on `My Profile` at the top right
2. Click on the `API Tokens` tab in the sidebar
3. Click `Create Token` and then use the `Edit zone DNS` template
4. In the permissions section, add `Zone:Zone:Read` and `Zone:DNS:Edit`
5. You can leave the Client IP Address Filtering section as-is
6. Select how long you'd like the token to be active for in the TTL section
7. Click `Continue to summary`
8. Verify everything and click `Create Token`
9. Copy the token and keep it saved, you will only see this once

### Running Nginx Proxy Manager

Once we've taken care of all the cloudflare shenanigans, we can start with Nginx Proxy Manager

1. Create a new directory to store the docker compose file and the data for the container
    ```
    $ mkdir -p nginx/data
    $ cd nginx
    ```

2. Create a new `docker-compose.yml` with the following content
    ```yaml
    services:
      app:
        container_name: nginx-proxy-manager
        image: 'jc21/nginx-proxy-manager:latest'
        restart: always
        ports:
          - '80:80'
          - '81:81'
          - '443:443'
        volumes:
          - ./data/data:/data
          - ./data/letsencrypt:/etc/letsencrypt
    ```

3. Start the docker container
    ```
    $ docker compose up -d
    ```

4. Access Nginx Proxy Manager

    Head over to <http://192.168.70.72:81> and continue with the user account creation. I trust that you'll be able to do this on your own!

### Setting up wildcard TLS certificates with Nginx Proxy Manager

After finishing up the basic set up with NPM from the previous steps, we can now work on getting TLS certificates using the DNS challenge.

1. Navigate to the `Certificates` tab in NPM
2. Click `Add Certificate -> Let's Encrypt via DNS`
3. For the `Domain Names` field, type `*.your-domain.com` and hit Enter
4. You can leave the `Key Type` as is
5. Select `Cloudflare` for `DNS Provider` field
6. Replace the dummy API Token with your own in the `Credentials File Content` field
7. Click `Save`

If all goes well, NPM will take a few seconds to get your shiny new TLS certificates for you!

### Configuring a service to use HTTPS

Now that we have the certificates we need, let's add jellyfin as a new proxy host to NPM so we can access it over HTTPS.
This is just an example, you'll have to adjust the values of the fields based on the service you're setting HTTPS up for.

1. Navigate to the `Hosts -> Proxy Hosts` tab in NPM
2. Click `Add Proxy Host`
3. For the `Domain Names` field, type `jellyfin.your-domain.com` and hit Enter
4. Enter `192.168.70.72` for the `Forward Hostname / IP` field
5. Enter `8096` for the `Forward Port` field
6. In some cases you might want to enable `Websockets Support`
7. Click on the `SSL` tab and select the wildcard certificate we created earlier in the `SSL Certificate` field
8. Click `Save`

Just like that, you can stream all of your legally acquired Linux ISOs over HTTPS

# Conclusion

I hope the above guide was helpful in getting you out of the stone ages. I realize that the tutorial may not fit every configuration out there,
but I have faith in you, the reader, that you'll be able to make it work for your own needs.

Let's hope that it won't take me another 3 years to write a new post.
