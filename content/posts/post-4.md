---
title: "Implementing domain based Multi-tenancy - 2"
date: 2022-06-05T00:01:58+05:30
description: "This is a tutorial on implementing a domain based multi-tenant application using NextJs and Openresty."
tags: [multi-tenant, "nextjs", "openresty"]
---

In the previous post I showed you how to set up a NextJs application to serve pages based on users custom domain.

**_If you haven't read the previous article follow here for [Part 1](https://hold7door.github.io/posts/post-3/). Otherwise please continue reading_**

The main problem now remains is to dynamically provision HTTPS certificates for the newly created domains so that data can be served over TLS. For this we use _Openresty_ which is wrapper around Nginx and provides inbuilt support for _Lua_ programming language.

Using openresty+Lua and other supporting modules we can create SSL certificates on demand from LetsEncrypt.

### Setting up Openresty

#### Step 1

If you have an existing installation of Nginx running we need to remove it first. Otherwise please move to Step 2.

_Warning: Please backup all your configuration files_.

To remove nginx -

1. First make sure you have made a backup of your nginx configuration files
2. Run the following commands to delete nginx and its related binaries

```
# sudo systemctl stop nginx.service
# sudo systemctl disable nginx.service
# sudo userdel -r nginx
# sudo rm -rf /etc/nginx
# sudo rm -rf /var/log/nginx
# sudo rm -rf /var/cache/nginx/
# sudo rm -rf /usr/lib/systemd/system/nginx.service
```

#### Step 2

<b>Now Install openresty:</b>

Prerequisites for installing Openresty:

```
> Debian and Ubuntu users

You're recommended to install the following packages using apt-get:

$ apt-get install libpcre3-dev \
    libssl-dev perl make build-essential curl

> Fedora and RedHat users

You're recommended to install the following packages using yum:

$ yum install pcre-devel openssl-devel gcc curl

```

This page from DigitalOcean does a good job on instructing how you can download and install Openresty -

[https://www.digitalocean.com/community/tutorials/how-to-use-the-openresty-web-framework-for-nginx-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-use-the-openresty-web-framework-for-nginx-on-ubuntu-16-04)

**Important: If you followed the above link, there are a few caveats that you need to take care of**

1. We need to build openresty with http2. So the command to configure and build should be: `./configure -j2 --with-pcre-jit --with-ipv6 --with-http_v2_module`. If its a success the output should be something like (check the paths to the nginx files and binaries)

```
nginx path prefix: "/usr/local/openresty/nginx"
nginx binary file: "/usr/local/openresty/nginx/sbin/nginx"
nginx modules path: "/usr/local/openresty/nginx/modules"
nginx configuration prefix: "/usr/local/openresty/nginx/conf"
nginx configuration file: "/usr/local/openresty/nginx/conf/nginx.conf"
nginx pid file: "/usr/local/openresty/nginx/logs/nginx.pid"
nginx error log file: "/usr/local/openresty/nginx/logs/error.log"
nginx http access log file: "/usr/local/openresty/nginx/logs/access.log"
nginx http client request body temporary files: "client_body_temp"
nginx http proxy temporary files: "proxy_temp"
nginx http fastcgi temporary files: "fastcgi_temp"
nginx http uwsgi temporary files: "uwsgi_temp"
nginx http scgi temporary files: "scgi_temp"

```

2. When setting up OpenResty as a Servie on _AWS AMI_ the `openresty.service` file should be created in `/usr/lib/systemd/system`

3. Openresty should run as _www-data_ user. Create user www-data if it does not exists using the following command

`$ sudo adduser --system --no-create-home --shell /bin/false --user-group www-data`

OpenResty workers should run as this user, so inside the nginx.conf file make sure you have this line

`user: www-data`

4. Create the directory _/var/log/openresty_ and make sure that inside the nginx.conf file this directory is set as the target for log files

Great! Now we have Openresty up and running. Now let's get to the good part and see how we can use the inbuilt Lua integration of Openresty for dynamically creating SSL certificates for custom domains.

### Installing LuaRocks and lua-resty-auto-ssl

The [lua-resty-auto-ssl](https://github.com/fffonion/lua-resty-openssl) plugin for OpenResty mentions itself as a -

```
 On the fly (and free) SSL registration and renewal inside OpenResty/nginx with Let's Encrypt.
```

This OpenResty plugin automatically and transparently issues SSL certificates from Let's Encrypt (a free certificate authority) as requests are received. It works like:

- A SSL request for a SNI hostname is received.
- If the system already has a SSL certificate for that domain, it is immediately returned (with OCSP stapling).
- If the system does not yet have an SSL certificate for this domain, it issues a new SSL certificate from Let's Encrypt. Domain validation is handled for you. After receiving the new certificate (usually within a few seconds), the new certificate is saved, cached, and returned to the client (without dropping the original request).

To install _lua-resty-auto-ssl_ plugin we first need to install _LuaRocks package manager_. For this we execute the following commands:

```
$ sudo apt install build-essential libreadline-dev unzip

$ wget https://luarocks.org/releases/luarocks-3.8.0.tar.gz

$ tar zxpf luarocks-3.8.0.tar.gz

$ cd luarocks-3.8.0

# Note that in this command line separators are important

$ ./configure --prefix=/usr/local/openresty/luajit \

 --lua-suffix=jit \

 --with-lua-include=/usr/local/openresty/luajit/include/luajit-2.1 \

--with-lua=/usr/local/openresty/luajit/

$ sudo ln -s /usr/local/openresty/luajit/bin/luarocks /usr/bin/luarocks

```

Now that we have LuaRocks installed we can use it to install the _lua-resty-auto-ssl_ plugin:

```
$ sudo luarocks install lua-resty-auto-ssl

$ sudo mkdir /etc/resty-auto-ssl

$ sudo chown www-data /etc/resty-auto-ssl

```

Cool! Now the final part remaining is to see how we can set up our nginx configuration to use this plugin to provision certificates.

### Configuring the Web Server

Inside the `http` block of our nginx.conf file we need to have the following lines -

```
http {
    # Conf for lua-auto-ssl
    # The "auto_ssl" shared dict should be defined with enough storage space to
    # hold your certificate data. 1MB of storage holds certificates for
    # approximately 100 separate domains.
    lua_shared_dict auto_ssl 2m;

    # The "auto_ssl_settings" shared dict is used to temporarily store various settings
    # like the secret used by the hook server on port 8999. Do not change or
    # omit it.
    lua_shared_dict auto_ssl_settings 64k;

    # A DNS resolver must be defined for OCSP stapling to function.
    #
    # This example uses Google's DNS server. You may want to use your system's
    # default DNS servers, which can be found in /etc/resolv.conf. If your network
    # is not IPv6 compatible, you may wish to disable IPv6 results by using the
    # "ipv6=off" flag (like "resolver 8.8.8.8 ipv6=off").
    resolver 8.8.8.8;

    # Initial setup tasks.
    init_by_lua_block {
        auto_ssl = (require "resty.auto-ssl").new()
        -- Define a function to determine which SNI domains to automatically handle
        -- and register new certificates for. Defaults to not allowing any domains,
        -- so this must be configured.
        auto_ssl:set("allow_domain", function(domain)
            -- ngx.log(ngx.STDERR, domain)
            -- Return false of restricted domains and IP address

            if string.find(domain, "www.example.com") then
                    return false
            end

            local http = require("resty.http")
            local httpc = http.new()
            httpc:set_timeout(3000)

            -- Make a API call to your web server for some custom logic to check whether to issue certificate for this domain or not
            local uri = "https://api.example.com/check-domain/" .. domain
            local res, err = httpc:request_uri(uri, {
                    ssl_verify = false,
                    method = "GET"
            })

            if not res then
                    return false
            end

            if res.status == 200 then
                    return true
            else
                    return false
            end

        end)

        -- This line should be present during development otherwise your account may hit rate limiting issues

        auto_ssl:set("ca", "https://acme-staging-v02.api.letsencrypt.org/directory")

        -- All certificates will be checked for renewal every 12 hrs
        -- Renewal happens if certificate expires in less than 30 days

        auto_ssl:set("renew_check_interval", 43200)

        auto_ssl:init()
    }

    init_worker_by_lua_block {
            auto_ssl:init_worker()
    }

    # Internal server running on port 8999 for handling certificate tasks.
    server {
    listen 127.0.0.1:8999;

    # Increase the body buffer size, to ensure the internal POSTs can always
    # parse the full POST contents into memory.
    client_body_buffer_size 128k;
    client_max_body_size 128k;

    location / {
        content_by_lua_block {
        auto_ssl:hook_server()
        }
    }
    }

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include conf.d/*.conf;
}
```

Note that we don't want to provision certificate for our own domain. Notice that we have done this for www.example.com which for example let's say is our own domain.

We also need to have a path in our configuration listening on port 80 for the ACME challenge from LetsEncrypt. So we need to set up a location block for this:

```
# Endpoint used for performing domain verification with Let's Encrypt.
location /.well-known/acme-challenge/ {
  content_by_lua_block {
    auto_ssl:challenge_server()
  }
}
```

And lastly but not the least, we also need to have a location block to proxy pass request to our NextJS application.

```
# NextJS to serve custom domain
location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
}
```

### Conclusion

Now any user who wants to map their custom domain to our application can add an **A record** and point it to our servers IP (which we need to provide them).

If SSL cert doesn't exist for a domain, it's generated and served automatically when you request the URL for the first time. Naturally, there is a latency of ~10s for the very first request. Next time onwards it's blazing fast since the certs are cached. Since LE certs are valid for 90 days, renewals happen automatically if the expiry date is < 30 days.

Some follow ups you might have like -

1. How to map CNAME ?
2. What to do in the case where we have multiple servers distributed accross the globe?

Connect me on twitter and I'll answer those. If needed I'll write another blog post about it in the future.

Hope you liked it, See you in the next blog :)
