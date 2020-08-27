# iwdlive.dev

The compose file `./idwlive-dev_postgres_www.yml` creates a common
network (`iwdlive-dev_common`) with postgres (`db`), nginx (`www`), frontend (`pwa`) and backend (`api`) containers.

## Postgres (db)
- Standard image (postgres) used.
- Opens port 5432 for 127.0.0.1
- Reads config from ./postgres.env
- Stores data to ./db/data/

## Nginx (www)
- Custom image based on nginx:alpine
- Uses ./www/nginx.conf and ./www/sites-enabled/*
- ./sites/ is mounted to /var/sites/ in the container
  (symlink build/ folders in ./sites/ to test builds)
- Stores cache files to ./cache/
- Uses SSL certificate from ./www/ssl/

## manage
Use `./manage mkcert <DOMAIN>` to generate a SSL certificate, stored in `./www/ssl/<DOMAN>`. This certificate will be valid for `<DOMAIN>` and `*.<DOMAIN>`.

All other parameters are forwarded to `docker-compose -f ./iwdlive-dev_postgres_www.yml`.

```bash
# Bring up containers
# [-d] detach from your terminal process
#      a/k/a daemonize
./manage [-d] up (api|db|pwa|www)

# Bring down (stop and destroy)
./manage down (api|db|pwa|www)

# Just stop
./manage stop (api|db|pwa|www)

# Bring up any combination of
# containers, separately

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
```

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

*No need to do any of this, if you are using [mkcert](#mkcert) and the installation didn't fail.*

Go to your browser settings (e.g. brave://settings/certificates) and add the mkcert `rootCA.pem` to your "Authorities". You can get its location with

```bash
mkcert -CAROOT
```

When asked, select the "identifying websites" option. The mkcert root certificate - which you use to sign the certificates for domains - is now in the list, named "org-mkcert development CA".
From now on, all certificates generated with mkcert are accepted by your browser.

Also have a look at the [advanced topics](https://github.com/FiloSottile/mkcert#advanced-topics) in the mkcert README!
