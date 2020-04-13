+++
title = "What I learned about SSL Certificates when building a Secured Kafka Cluster"
date = "2020-04-07"
tags = [ "SSL", "Kafka", "Certificates" ]
categories = [ "Apache Kafka", "Cluster" ]
+++

To build a multi-protocol Apache Kafka Clusters to allow for __SSL Client Authentication__ with
__PLAINTEXT__ for inter broker communication, I needed to generate both broker and client SSL certificates.
There were many interesting things I learned in this process and wanted to share them.

An upcoming article will be using these certificates for setting up my Secured Apache Kafka Cluster.

### Disclaimer

I build certificates to explore the security available in various systems; in this particular scenario, it is
to explore a mixed protocol Kafka Cluster. Please seek advice from your security teams when creating certificates. 

## __Introduction__

If you only need to create certificates as part of an enterprise setup, consider actively supported
projects such as Confluent's [Ansible Playbooks](https://docs.confluent.io/current/installation/cp-ansible/index.html).

There are excellent articles on SSL Certificates, including Confluent's [documentation](https://docs.confluent.io/current/kafka/authentication_ssl.html)
and an excellent [tutorial](https://docs.confluent.io/current/security/security_tutorial.html#generating-keys-certs).
The documentation and tutorial focuses on __SASL__ client-side authentication (__SASL_SSL__ protocol) 
and I uncovered some unique challenges with client-side __SSL__ authentication.

My experience in certificate generation is based on using __OpenSSL__. There are other means of generating certificates,
so if another means is used for certificate creation, some of these highlights may not apply to you.

My full scripts are in the repository [Kafka SSL Cluster](https://github.com/nbuesing/kafka-ssl-cluster).

## __Naming__

Your certificate __subject__ must start with /, and should contain at minimum a __CN__ element, as in __/CN=${hostname}__.

For broker certificates, have the __CN__ matches the hostname, and then leverage [Subject Alternate Names](#SAN) for
ensuring the certificate can be used with and without domain name successfully.

There are no wildcard options with the standard Apache Kafka Authorizer, `kafka.security.authorizer.AclAuthorizer`.
So if you a complete subject of common name (CN), organization unit (OU), organization (OU), locality (L), state (S),
and country (C) the full name is what will be used as the User in ACLs.

This: `User:CN=broker-1,OU=KAFKA,O=COMPANY,L=CITY,ST=MN,C=US`

Not This: `User:CN=broker-1`

## __JKS key and keystore passwords should be the same__

When creating Java key-stores (.jks files) for your JVMs, make sure the keystore password is the same as the key password.
This is due to limitations of the __SunX509 KeyManagerFactory__.  Java Secure Socket Extension (JSSE) [Reference Guide](https://docs.oracle.com/en/java/javase/11/security/java-secure-socket-extension-jsse-reference-guide.html#GUID-65A7A023-AE02-4A95-8210-386AE6F18EB5).

> All keys in the KeyStore must be protected by the same password 

I have found a few other documented cases of this with other systems, the most recent is with [BitBucket](https://confluence.atlassian.com/bitbucketserverkb/bitbucket-server-fails-to-start-with-ssl-java-security-unrecoverablekeyexception-cannot-recover-key-814205872.html).

## __Extensions__

Be sure the certificates have the proper x509 extensions.

### __Basic Contraints extension for CA certificate__

```
basicConstraints=CA:TRUE,pathlen:0
```

If you setup SSL certificates for client authentication, the CA certificate needs the following x509 extension indicating it is
indeed a CA certificate. Without this extension, it will not authenticate client certificates.


For additional information, see [x509v3 config](https://www.openssl.org/docs/man1.0.2/man5/x509v3_config.html).

### __Subject Alternate Name Extension for Brokers__ {#SAN}

```
subjectAltName=DNS:${i}, DNS:${i}.${DOMAIN}
```

When it comes to SSL Certificates for encryption, it is important that it can have extensions to handle being used with and without
domain names.

### __Extended Key Usage Extension (for all)__ {#EKU}

```
extendedKeyUsage=serverAuth,clientAuth
```

The public key a certificate had to be noted that it can be used for server and client authentication. For my cluster,
I configured all certificates so they can be used for both server and client authentication.  I added both to all 
certificates.

## __extensions are not copied from request to certificate__ {#copyExtensions}

By default, extensions added to a request is not propagated to the certificate.  In order for extensions to be copied, 
you would need `copy_extensions = copy` added to your `openssl.cfg`. I was unsuccessful in getting this to work in 
MacOS so I updated my scripts to explicitly add extensions when certificate requests are signed. 

It is a good reminder if you create certificate requests for another group to sign, be sure to indicate extensions you
need in your certificates.

## __Check Status after every call to OpenSSL__

If you write scripts, make sure you check return status codes after each command and fail on error.  I have spent many hours
troubleshooting script errors only to find out that something failed much earlier in the process.

After each command, add the following. Scripts are finicky, especially with loops. So double-check the error handling 
works as well. This is the type of check I do after every `openssl` and every other command.
 
```
[ $? -eq 1 ] && echo "failure" && exit
```

## __Intermediate Certificates__

Most enterprises do not sign machine or applications certificates with their top-level CA certificate; they use an 
intermediate certificate. Using an intermediate certificate requires a certificate chain when creating the trust-store. 
Personally, I use intermediate certificates, so it is a reminder on how to properly chain them when creating the pkcs12 file.

##  __The openSSL configuration file__

The configuration file of OpenSSL, __openssl.cnf__ can vary greatly between Unix systems and even between Linux distributions.

For every command of __openssl__ that uses `openssl.cfg` provide a configuration file; otherwise, a default file will be used
and it varies greatly between OS distributions. To avoid confusing on how __OpenSSL__ executes your request, I explicitly
provide the `openssl.cfg` to use. I keep this file with my scripts. Furthermore, if you are scripting your certificate
process, leverage inline files to provide a custom configuration file to each specific command execution; it is then
self-documenting too.

An example of inlining a file concatenation.
```
-config <(cat ./openssl.cnf <(printf "\n[ext]\nbasicConstraints=CA:TRUE,pathlen:0"))
```

## __Check your OpenSSL documentation__

OpenSSL varies between Unix variants and even between Linux distributions; please read the manual pages for your specific
version and release of `openssl`. Also, there could be a bug or limitation with the instance of  `openssl` you are using.
For example, I was not successful leveraging [copy_extensions](#copyExtensions) on MacOS.

## __Conclusion__

What I have documented here were the major pieces I had to figure out when building certificates for client authentication
for my Kafka Cluster. It wouldn't surprise me if there are other things I could uncover, especially limiting extensions
only between the broker certificates and client certificates.
 
If you want a glimpse to the scripts that I will use in part 2 of this blog where I configure my Secured (Mix-Protocol)
Kafka Cluster, you find them all at [Kafka SSL Cluster](https://github.com/nbuesing/kafka-ssl-cluster).

