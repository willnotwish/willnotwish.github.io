<!-- 2017-04-10-using-certbot.md -->

I used letsencrypt (or is it certobt?) to generate an SSL certificate for The Running Curve's promotional website.

Those certificates only last for three months, and then they need renewing. It's not that difficult, but in all the rush to get theat site ready I didn't make any notes and then I forgot how to renew it.

The concept is simple enough: tell the issuing authority which domain you want to renew, put a file in a "well known" place on the server to prove ownership (which only I can do as I'm the one who administers the domain in question), and tell letsencrypt's CA servers to verify that it's there. letsencrypt then issues the certificate and puts it on my server in the correct place (somewhere where nginx can find it).

The documentation and hearsay about this is very confusing.

For a start, the letsencrypt client is called "certbot".

The main reference is here https://certbot.eff.org/#ubuntuother-nginx

Very briefly:
1. As a user with sudo privileges ('nick')
2. install the certbot-auto script
3. Make it executable: chmod a+x certbot-auto
4. Run it: ./certbot-auto certonly
5. Follow the instructions using the "webroot" method
6. Enter the domain name you want to renew
7. Enter the webroot: /var/www/letsencrypt

Then, certbot will generate a file (x) and put it in that webroot. The CA's server will try to retrieve the URL 
http://therunnningcurve.com/.well-known/acme-challenge/x

You have to make sure nginx can serve files from that location. In my nginx.conf file under therunningcurve.com domain section I have this block:

```
      location ^~ /.well-known/acme-challenge/ {

        # Set correct content type. According to this:
        # https://community.letsencrypt.org/t/using-the-webroot-domain-verification-method/1445/29
        # Current specification requires "text/plain" or no content header at all.
        # It seems that "text/plain" is a safe option.
        default_type "text/plain";

        # This directory must be the same as in /etc/letsencrypt/cli.ini
        # as "webroot-path" parameter. Also don't forget to set "authenticator" parameter
        # there to "webroot".
        # Do NOT use alias, use root
        root         /var/www/letsencrypt;
      } 
```

That's it.
