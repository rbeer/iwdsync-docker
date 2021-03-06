# iwdlive.dev

***Quickstart:***
*[Install mkcert](https://github.com/FiloSottile/mkcert#installation), [node.js](https://nodejs.org/en/download/) and run `./manage init`*

---
The compose file `./idwlive-dev_postgres_www.yml` creates a
network (`iwdlive-dev_common`) with postgres (`db`), redis (`redis`),
nginx (`www`), frontend (`pwa`) and backend (`api`) containers.

| iwdlive-db |  |
|-:|-|
| image | postgres |
| ports | 127.0.0.1:5432:5432 |
| env/conf | postgres.env |
| **iwdlive-www** |  |
| image | www/. (nginx:alpine) |
| ports | 127.0.0.1:80:80 (http)<br>127.0.0.1:443:443 (https) |
| env/conf | www/nginx.conf<br>www/sites-/\*<br>www/ssl/\*\*/(key\|cert).pem |
| **iwdlive-redis** |  |
| image | redis:latest |
| env/conf | redis.conf |
| **iwdlive-api** |  |
| image | ../iwdsync-backend/. (python:3-alpine) |
| env/conf | ../iwdsync-backend/.env.docker |
| **iwdlive-pwa** |  |
| image | ../iwdsync/. (node:lts-alpine) |
| env/conf | ../iwdsync/.env.docker |

## manage
`./manage init` runs all steps necessary to get a demo started.

Use `./manage mkcert <DOMAIN>` to generate a SSL certificate, stored in `./www/ssl/<DOMAN>`.
This certificate will be valid for `<DOMAIN>` and `*.<DOMAIN>`.

All other parameters are forwarded to `docker-compose -f ./iwdlive-dev_postgres_www.yml`.

```bash
# Bring up containers
# [-d] detach from your terminal process
./manage up [-d] (api|db|pwa|www)

# Stop a service
./manage stop (api|db|pwa|www)

# Bring all services down (stop and destroy)
# This will wipe your DB!
./manage down

# Execute command inside of running container
./manage exec (api|db|pwa|www) <CMD>
# e.g. reload nginx configs
./manage exec www nginx -s reload
```

## SSL certificate

### mkcert
I recommend using [mkcert](https://github.com/FiloSottile/mkcert) to manage all your development SSL certificate needs.

[Install mkcert](https://github.com/FiloSottile/mkcert#installation) and generate/install the CA certificate.
You probably want to restrict the automatic installation to browsers, setting `$TRUST_STORES` to `nss`.

```bash
TRUST_STORES=nss mkcert -install
```

To finish the SSL setup, create a certificate for `iwdlive.dev`

```bash
mkcert \
  -key-file ./www/ssl/iwdlive.dev/key.pem \
  -cert-file ./www/ssl/iwdlive.dev/cert.pem \
  iwdlive.dev *.iwdlive.dev

# or use ./manage
./manage mkcert iwdlive.dev
```

You can use any other way to generate your cert/key pair. Just copy both .pem
files somewhere into `./www/ssl/` and reference them in your host config.

The server is already set up for `iwdlive.dev`. The next section describes how
to add new domains.

### Server

Suppose you want to add the domain `streamsync.dev`.

Start with generating the SSL cert and key

```bash
./manage mkcert streamsync.dev
```

The path `./www/ssl` is mounted to `/etc/nginx/ssl` and
can be referenced in configs relative to `/etc/nginx`.

For our example:

```bash
# The certificate you just created...
<project_root>/docker/www/ssl/streamsync.dev/cert.pem

# is mounted inside the container...
/etc/nginx/ssl/streamsync.dev/cert.pem

# and can be used in host configs with a relative path
ssl_certificate ssl/streamsync.dev/cert.pem;
```

Now copy `./www/sites-enabled/iwdlive.dev` to `./www/sites-enabled/streamsync.dev`
and change the paths to `ssl_certificate` and `_key`, as well as `$DOMAIN` and `$server_name`.

```nginx
  server {
    set $DOMAIN 'streamsync.dev';
    # dont't change this!
    # those are name and port of the frontend container
    # on the internal docker network
    set $DEV_SERVER_ADDRESS 'iwdlive-pwa';
    set $DEV_SERVER_PORT '3000';

    server_name streamsync.dev;
    listen 80;
    listen 443 ssl; # remove the default_server, here
    ssl_certificate ssl/streamsync.dev/cert.pem;
    ssl_certificate_key ssl/streamsync.dev/key.pem;

    ...
  }
```

All that's left is to reload the nginx configs

```bash
./manage exec www nginx -s reload
```

### Client

***No need to do any of this, if you are using [mkcert](#mkcert) and the installation didn't fail.***

Go to your browser settings (e.g. brave://settings/certificates) and add the mkcert `rootCA.pem`
to your "Authorities". You can get its location with

```bash
mkcert -CAROOT
```

When asked, select the "identifying websites" option. The mkcert root certificate - which you
use to sign the certificates for domains - is now in the list, named "org-mkcert development CA".
From now on, all certificates generated with mkcert are accepted by your browser.

Also have a look at the [advanced topics](https://github.com/FiloSottile/mkcert#advanced-topics) in the mkcert README!
