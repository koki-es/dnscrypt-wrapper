# Name

dnscrypt-wrapper - A server-side dnscrypt proxy.

[![Build Status](https://travis-ci.org/cofyc/dnscrypt-wrapper.png?branch=master)](https://travis-ci.org/cofyc/dnscrypt-wrapper)

## Table of Contents

* [Description](#description)
* [Installation](#installation)
* [Usage](#usage)
  * [Quick start](#quick-start)
  * [Running unauthenticated DNS and the dnscrypt service on the same port](#running-unauthenticated-dns-and-the-dnscrypt-service-on-the-same-port)
  * [Key rotation](#key-rotation)
* [Chinese](#chinese)
* [See also](#see-also)

## Description

This is dnscrypt wrapper (server-side dnscrypt proxy), which helps to
add dnscrypt support to any name resolver.

This software is modified from
[dnscrypt-proxy](https://github.com/jedisct1/dnscrypt-proxy).

## Installation

Install [libsodium](https://github.com/jedisct1/libsodium) and [libevent](http://libevent.org/) 2.1.1+ first.

On Linux:

    $ ldconfig # if you install libsodium from source
    $ git clone --recursive git://github.com/cofyc/dnscrypt-wrapper.git
    $ cd dnscrypt-wrapper
    $ make configure
    $ ./configure
    $ make install

On FreeBSD:

    $ pkg install dnscrypt-wrapper

On OpenBSD:

    $ pkg_add -r gmake autoconf
    $ pkg_add -r libevent
    $ git clone --recursive git://github.com/cofyc/dnscrypt-wrapper.git
    $ cd dnscrypt-wrapper
    $ gmake LDFLAGS='-L/usr/local/lib/' CFLAGS=-I/usr/local/include/

On MacOS:

    $ brew install dnscrypt-wrapper

In Docker:

    See https://github.com/jedisct1/dnscrypt-server-docker.

## Usage

### Quick Start

1) Generate the provider key pair:

```
$ dnscrypt-wrapper --gen-provider-keypair
```

This will create two files in the current directory: `public.key` and
`secret.key`.

This is a long-term key pair that is never supposed to change unless the
secret key is compromised. Make sure that `secret.key` is securely
stored and backuped.

If you forgot to save your provider public key:

```
$ dnscrypt-wrapper --show-provider-publickey --provider-publickey-file <your-publickey-file>
```

This will print it out.

2) Generate a time-limited secret key, which will be used to encrypt
and authenticate DNS queries. Also generate a certificate for it:

```
$ dnscrypt-wrapper --gen-crypt-keypair --crypt-secretkey-file=1.key
$ dnscrypt-wrapper --gen-cert-file --crypt-secretkey-file=1.key --provider-cert-file=1.cert \
                   --provider-publickey-file=public.key --provider-secretkey-file=secret.key
```

In this example, the time-limited secret key will be saved as `1.key`
and its related certificate as `1.cert` in the current directory.

Time-limited secret keys and certificates can be updated at any time
without requiring clients to update their configuration.

NOTE: By default, secret key expires in 1 day (24 hours) for safety. You can
change it by adding `--cert-file-expire-days=<your-expected-expiraiton-days>`,
but it's better to use short-term secret key and use
[key-rotation](#key-rotation) mechanism.

3) Run the program with a given key, a provider name and the most recent certificate:

```
$ dnscrypt-wrapper --resolver-address=8.8.8.8:53 --listen-address=0.0.0.0:443 \
                   --provider-name=2.dnscrypt-cert.<yourdomain> \
                   --crypt-secretkey-file=1.key --provider-cert-file=1.cert
```

The provider name can be anything; it doesn't have to be within an existing
domain name. However, it has to start with `2.dnscrypt-cert.`, e.g.
`2.dnscrypt-cert.example.com`.

When the service is started with the `--provider-cert-file` switch, the
proxy will automatically serve the certificate as a TXT record when a
query for the provider name is received.

As an alternative, the TXT record can be served by a name server for
an actual DNS zone you are authoritative for. In that scenario, the
`--provider-cert-file` option is not required, and instructions for
Unbound and TinyDNS are displayed by the program when generating a
provider certificate.

You can get instructions later by running:

```
$ dnscrypt-wrapper --show-provider-publickey-dns-records
                   --provider-cert-file <path/to/your/provider_cert_file>
```

4) Run dnscrypt-proxy to check if it works:

```
$ dnscrypt-proxy --local-address=127.0.0.1:55 --resolver-address=127.0.0.1:443 \
                 --provider-name=2.dnscrypt-cert.<yourdomain> \
                 --provider-key=<provider_public_key>
$ dig -p 55 google.com @127.0.0.1
```

`<provider_public_key>` is public key generated by `dnscrypt-wrapper --gen-provider-keypair`, which looks like `4298:5F65:C295:DFAE:2BFB:20AD:5C47:F565:78EB:2404:EF83:198C:85DB:68F1:3E33:E952`.

Optionally, add `-d/--daemonize` flag to run as a daemon.

Run `dnscrypt-wrapper -h` to view command line options.

### Running unauthenticated DNS and the dnscrypt service on the same port

By default, and with the exception of records used for the
certificates, only queries using the DNSCrypt protocol will be
accepted.

If you want to run a service only accessible using DNSCrypt, this is
what you want.

If you want to run a service accessible both with and without
DNSCrypt, what you usually want is to keep the standard DNS port for
the unauthenticated DNS service (53), and use a different port for
DNSCrypt. You don't have to change anything for this either.

However, if you want to run both on the same port, maybe because only
port 53 is reachable on your server, you can add the `-U`
(`--unauthenticated`) switch to the command-line. This is not
recommended.

### Key rotation

Time-limited keys are bound to expire.

`dnscrypt-proxy` can check if the current key for a given server is
not going to expire soon:

```
$ dnscrypt-proxy --resolver-address=127.0.0.1:443 \
                 --provider-name=2.dnscrypt-cert.<yourdomain> \
                 --provider-key=<provider_public_key> \
                 --test=10080
```

The `--test` option is followed by a "grace margin".

The command will immediately exit after verifying the certificate validity.

The exit code is `0` if a valid certificate can be used, `2` if no valid
certificates can be used, `3` if a timeout occurred, and `4` if a currently
valid certificate is going to expire before the margin.

The margin is always specified in minutes.

This can be used in a cron tab to trigger an alert before a key is
going to expire.

In order to switch to a fresh new key:

First, create a new time-limited key (do not change the provider key!) and
its certificate:

```
$ dnscrypt-wrapper --gen-crypt-keypair --crypt-secretkey-file=2.key
$ dnscrypt-wrapper --gen-cert-file --crypt-secretkey-file=2.key --provider-cert-file=2.cert \
                   --provider-publickey-file=public.key --provider-secretkey-file=secret.key \
                   --cert-file-expire-days=1
```

Second, Tell new users to use the new certificate but still accept the old
key until all clients have loaded the new certificate:

```
$ dnscrypt-wrapper --resolver-address=8.8.8.8:53 --listen-address=0.0.0.0:443 \
                   --provider-name=2.dnscrypt-cert.<yourdomain> \
                   --crypt-secretkey-file=1.key,2.key --provider-cert-file=2.cert
```

Note that both `1.key` and `2.key` have be specified, in order to
accept both the previous and the current key.

Third, Clients automatically check for new certificates every hour. So,
after one hour, the old certificate can be refused, by leaving only
the new one in the configuration:

```
$ dnscrypt-wrapper --resolver-address=8.8.8.8:53 --listen-address=0.0.0.0:443 \
                   --provider-name=2.dnscrypt-cert.<yourdomain> \
                   --crypt-secretkey-file=2.key --provider-cert-file=2.cert
```

Please note that on Linux systems (kernel >= 3.9), multiples instances of
`dnscrypt-wrapper` can run at the same time. Therefore, in order to
switch to a new configuration, one can start a new daemon without
killing the previous instance, and only kill the previous instance
after the new one started.

This also allows upgrades with zero downtime.

## Chinese

- CentOS/Debian/Ubuntu 下编译 dnscrypt-wrapper: http://03k.org/centos-make-dnscrypt-wrapper.html
- dnscrypt-wrapper 使用方法: http://03k.org/dnscrypt-wrapper-usage.html

注：第三方文档可能未及时与最新版本同步，以 README.md 为准。

## See also

- https://dnscrypt.org/
- https://github.com/jedisct1/dnscrypt-proxy
- https://github.com/cofyc/dnscrypt-wrapper
