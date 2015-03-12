# Certify
Certify creates and self-signs X.509 SSL/TLS certificates with the "subjectAltName" extension.

## Introduction

**First things first: if you want a SSL/TLS certificate for use on a public-facing https website, go to one of the [many vendors](https://www.sslshopper.com/ssl-certificate-list.html) and buy one. That is currently the only way to reliably authenticate and encrypt your website for your visitors.**

The actual security of the web's Public Key Infrastructure is [being heavily criticised](http://www.thoughtcrime.org/blog/ssl-and-the-future-of-authenticity/). Some even claim that [OpenSSH is written by monkeys](https://www.peereboom.us/assl/assl/html/openssl.html). But until [something better comes along](https://letsencrypt.org/) you'll just have to swallow it and pay money to the big boys to be protected.

With all that said, for small, personal projects [it's better to have a self-signed certificate than no certificate at all](http://serverfault.com/questions/279780/is-a-self-signed-ssl-certificate-a-false-sense-of-security). Once your browser has a legit copy of the self-signed certificate, you can securely access the website over an encrypted TLS connection. With governments controlling root certificate authorities, some even argue that a securely distributed self-signed certificate is [more secure than a paid certificate](http://security.stackexchange.com/questions/42409/are-self-signed-certificates-actually-more-secure-than-ca-signed-certificates-now). At a mininum, a self-signed certificate successfully avoids sending passwords in clear text.

## Self-signing using `openssl`

The basics of self-signing certificates are pretty easy. The standard tool to create certificates is the `openssl` command line utility. First, you need a public/private keypair, which you can create with `openssl genpkey` or `openssl genrsa`. Then, you use this key to create a certificate request with `openssl req`. Finally, you sign the request with either `openssl x509` or `openssl ca`. As you can see, [there isn't one single way to create a certificate with openssl](http://stackoverflow.com/questions/21297139/how-do-you-sign-openssl-certificate-signing-requests-with-your-certification-aut). More information about these commands can be found in their respective man pages: `man genpkey`, `man req`, `man x509`, etc.

Fortunately, openssl offers a possibility to do *all the previous steps at once*. I found this out by carefully reading the man pages, because the information on the internet about generating certificates is generally contradictory, outdated [and sometimes even dangerous](http://wingolog.org/archives/2014/10/17/ffs-ssl). To create a private key and a self-signed certificate at once, issue the following command:

    openssl req -x509 -new -keyout myserver.key -out myserver.crt

This command will ask you to input a few things like an "Organization Name" and most importantly a "Common Name", where you should enter the domain name of the website this certificate will be used for. You can safely leave the other fields empty, because client software never checks them anyway. You can inspect the resulting certificate with the command `openssl x509 -in myserver.crt -noout -text`.

## Downsides

Generating a certificate with the previous command has two downsides:

1. The private key -- stored in `myserver.key` -- will be encrypted with a passphrase. This means you cannot use it on an unattended webserver like [Apache](https://httpd.apache.org/) or [nginx](http://nginx.org/), because it will ask for the passphrase everytime you restart the webserver. You can solve this problem by decrypting the key with `openssl pkey -in myserver.key -out myserver.decrypted.key`

2. Depending on your operating system's configuration of the OpenSSl library, the resulting certificate will almost certainly lack the "subjectAltName" extension. This extension makes it possible to have one certificate that is valid for multiple domain names. This is, however, very convenient when configuring a webserver.

## Webserver configuration

A typicial nginx configuration for serving a website will involve two `server` directives. One matches example.com and simply redirects all traffic to www.example.com. The other matches www.example.com and actually serves the web pages. It's [important to have one canonical domain name](http://www.sitepoint.com/domain-www-or-no-www/), because otherwise the Google might (rightfully) accuse you of hosting the same content on multiple domain names. Here is an example of the right thing to do:

    server {
      server_name www.example.com;
      listen 80;
      listen 443 ssl;
      ssl_certificate myserver.crt;
      ssl_certificate_key myserver.key;
      return 301 $scheme://example.com$request_uri;
    }

    server {
      server_name example.com;
      listen 80;
      listen 443 ssl;
      ssl_certificate myserver.crt;
      ssl_certificate_key myserver.key;
      location / {
        root /var/www/;
      }
    }

This setup will redirect all traffic for **www.example.com** to **example.com**.
There is a problem however, and that is that the certificate `myserver.crt` can only contain one domain name in the Common Name field. When that name is www.example.com, the first `server` block wil serve an invalid certificate. When that name is example.com, the second `server` block will serve an invalid certificate. To solve this, you will have to generate two separate certificates, one with www.example.com as the Common Name, and one with example.com as the Common Name.

## SubjectAltName

SubjectAltName to the rescue! The SubjectAltName extension has been standardized in 2002 by [RFC 3280](https://tools.ietf.org/html/rfc3280). The extensions allows for multiple DNS domain names to be included in a certificate. By defining both example.com and www.example.com as the alternative names, you can use the same certificate for both domains. Creating a certificate with the subjectAltName extension is a little tricky, and the [many tutorials](https://duckduckgo.com/?q=create+subjectaltname+self-signed+certificate&t=debian) on the internet all instruct you to edit the openssl.cnf file manually for each certificate that you create.

## Certify command line utility

This is, finally, where Certify comes into play! It's a dead-simple command-line utility to create self-signed certificates for multiple domains without touching any config files. You use it as follows:

    ./certify example.com [www.example.com] [mail.example.com] [...]

Simply pass the alle the domain names as arguments to the `certify` command. After answering the few questions that openssl asks you, you will find a self-signed certificate and the accompanying private key in the current directory. The certificate will be valid for 10 years and uses 2048 bit encryption. Both these values can easily be changed in the source code of the `certify` script.

## Wildcard certificates

You can also use Certify to create [wildcard certificates](https://en.wikipedia.org/wiki/Wildcard_certificate). By default, the `certify` command uses the first argument to determine the file name for the new certificate, so you will have to pass the wildcard as the second (or higher) argument. Here is an example that generates a wildcard certificate:

    ./certify example.com *.example.com'

You can use the resulting certificate on example.com and on any subdomain of example.com.

## License

Everything in this repository is freely available under a version of the [GPL](https://gnu.org/licenses/gpl.html) of your choosing.
