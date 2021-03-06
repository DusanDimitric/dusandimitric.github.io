---
layout: post
title:  "Sharing Secrets Securely"
description: "Brief overview of how to securely send secret files"
tags: [UNIX, Encryption, Credentials]

---

As a developer|sysadmin|devops or almost any other kind of IT person you are
going to need to share some secret information with your colleagues at some
point. Think passwords, DB credentials, AWS credentials... extremely sensitive
stuff.

Below are a couple of my favourite ways to do it, ordered from least secure and
easiest to use to most secure and "hardest" to wrap your head around and use.
They are all UNIX/terminal based since I have literally never seen GUI in my
life :)

First off, let's create a playground directory and a fake secret file:

```bash
$ cd ~
$ mkdir secret_sharing/
$ cd secret_sharing/
$ cat << _EOF_ >> ./secret.txt
USER: dusan
PASS: password123
_EOF_
$ ls
secret.txt
```

## Content:
1. [Zip](#1-zip)
2. [ccrypt](#2-ccrypt)
3. [OpenSSL/LibreSSL](#3-openssllibressl)
4. [GnuPG](#4-gnupg)
5. [Troubleshooting: GnuPG Non-ASCII User Identifier](#troubleshooting-gnupg-non-ascii-user-identifier)
6. [Bonus 1: GnuPG Key Management](#bonus-1-gnupg-key-management)
7. [Bonus 2: GnuPG Signatures](#bonus-2-gnupg-signatures)
8. [Bonus 3: GnuPG Public Key Verification & _Web of Trust_](#bonus-3-gnupg-public-key-verification--web-of-trust)
9. [Pushing Privacy Further - Final Tips](#pushing-privacy-further---final-tips)

# 1. Zip

`zip` is a utility that comes with most popular UNIX derivatives, so there
is most likely no need to even install anything. You can check if you have
it by running:

```bash
zip --version
```

In order to transfer the secret file securely to another person we can `zip`
compress it and encrypt the compressed file with a simple password:

```bash
$ zip -e secret.zip secret.txt
Enter password:
Verify password:
  adding: secret.txt (stored 0%)
$ ls
secret.txt secret.zip
```

Remove the original secret file, since we are going to derive it from the
zip package:
```bash
$ rm secret.txt
```

Now let's reverse the process and unzip/decrypt:
```bash
$ unzip secret.zip
Archive:  secret.zip
[secret.zip] secret.txt password:
 extracting: secret.txt
$ ls
secret.txt secret.zip
```

And remove the zip file in order to clean up:
```bash
$ rm secret.zip
```

That's as simple as it gets.

## UX and Security

`zip` is extremely easy to use and in most cases requires no installation or
setup.

There are a couple of problems with this method though. Everyone can see the
names of the files inside the encrypted `zip` archive. This is an unnecessary
information leak. Choose your filenames wisely if you wish to use this method.

There are also problems inherent to all symmetric encryption schemes. Using
symmetric encryption only works well if the person that you want to share the
secret file with is physically near you so you can verbally share the decryption
password with them.

You can also use this method to share secret files with people who are not in
your physical vicinity - by sending them the password over some different
channel than the one that you used to send the encrypted file to them.

Let's say that you have sent the compressed/encrypted file to the recipient in
an Email attachment. There are a couple of ways to pretty securely send the
decryption password, here are 2 ideas:

1. Call them using some E2E encrypted VoIP application and tell them the password
2. Send the password using some E2E encrypted chat application, preferably with message auto-expiry

One of the big security risks with symmetric, password based secret sharing are
keyloggers and audio surveillance (smartphones, laptop or hidden mics), so you
should keep in mind that this is not the best way to share information that is
a matter of life or death.

## Verdict:
```
Security : 5
UX       : 10
```

# 2. ccrypt

`ccrypt` is a nice and easy tool designed for secret sharing.

Go to http://ccrypt.sourceforge.net/ and install `ccrypt` on your machine if
you don't have it already.

To encrypt, use `ccencrypt` and enter an encryption password which will be
used for decryption later:
```bash
$ ccencrypt ./secret.txt
Enter encryption key:
Enter encryption key: (repeat)
$ ls
secret.txt.cpt
```

Once you encrypt the file, the original is rewritten by the newly created
encrypted file - `secret.txt.cpt`.

In order to decrypt - use `ccdecrypt` which will prompt you for the password
that you entered during encryption.

```bash
$ ccdecrypt secret.txt.cpt
Enter decryption key:
$ ls
secret.txt
```

## UX and Security

`ccrypt` is extremely simple and easy to set up and use.

It is more secure than `zip` compression because of the fact that it overwrites
the original file on disk, leaving no way to recover it afterwards.

Aside from that - it shares the same security challenges as the previous method
since we're again dealing with passwords.

## Verdict:
```
Security : 6.5
UX       : 10
```

# 3. OpenSSL/LibreSSL

**OpenSSL** has received some pretty negative publicity in the past because of
its shortcomings (see - [heartbleed](http://heartbleed.com/)). The programmers
who worked on it dispersed which made the **OpenSSL** development stop. That's
why we will use **LibreSSL** instead.

As [the LibreSSL website](https://www.libressl.org/) states:

> LibreSSL is a version of the TLS/crypto stack forked from OpenSSL in 2014, with
goals of modernizing the codebase, improving security, and applying best
practice development processes.

## Installation

**OpenSSL** comes with most UNIX-like derivatives, and some of them have upgraded
to **LibreSSL**.

The [LibreSSL GitHub README](https://github.com/libressl-portable/portable#compatibility-with-openssl) says:

> LibreSSL is API compatible with OpenSSL 1.0.1, but does not yet include all
new APIs from OpenSSL 1.0.2 and later. LibreSSL also includes APIs not yet
present in OpenSSL. The current common API subset is OpenSSL 1.0.1.

What this means, for the purpose of this article, is that you can use either
**OpenSSL** or **LibreSSL**. Commands I use below are the same for both.

If neither one of those are present on your system, the installation is pretty
straight-forward for both, so pick one (**LibreSSL** if you can't decide) and
let's get on with it.

## Symmetric key use case

Let's start with the simplest possible example, using the AES-256 cypher.

```bash
$ openssl enc -aes-256-ctr -in secret.txt -out secret.txt.enc
enter aes-256-ctr encryption password:
Verifying - enter aes-256-ctr encryption password:
```

We now have an encrypted file that we can send to the recipient.
```bash
$ ls
secret.txt     secret.txt.enc
```

Notice that the original file is still there. Let's ([naively](https://ssd.eff.org/en/module/how-delete-your-data-securely-linux))
delete it:
```bash
$ rm secret.txt
```

Use the `-d` flag to decrypt and enter the decryption password:
```bash
$ openssl enc -aes-256-ctr -d -in secret.txt.enc -out secret.txt
enter aes-256-ctr decryption password:
$ cat secret.txt
USER: dusan
PASS: password123
```

This is as simple as it gets and it's not much more secure than using `zip`.

## Asymmetric key use case

### Generating an asymmetric key pair

**Up until now we have been dealing with `symmetric` encryption - meaning that
both parties share the same key, which is used both for encryption and
decryption.**

**From this point forward, we are going to explore `asymmetric` encryption,
meaning that there are 2 keys involved - one for encryption (public key) and
the other one for decryption (private key).**

> The reverse is also common - using the private key to encrypt and the public
key to decrypt, but that's not relevant for our particular use case here.

Let's start by generating the **private/public** key pair.

I'll generate an **RSA 4096 bit** private key first:
```bash
$ openssl genrsa -out privkey.pem 4096
```

**NOTICE:** The private key should be kept in a very safe place! Never share it
with anyone. It should absolutely always stay only on your machine. I will go
one step forward and encrypt the private key itself with a password:
```bash
$ openssl rsa -in privkey.pem -aes-256-ctr -out privkey.pem.enc
```

Delete the unencrypted private key ([naively](https://ssd.eff.org/en/module/how-delete-your-data-securely-linux)):
```bash
$ rm privkey.pem
```

Now, for the second part, I derive the **public key** from the previously created
and encrypted **private key**:
```bash
$ openssl rsa -in privkey.pem.enc -pubout -out pubkey.pem
```

### Encryption/Decryption

If you want someone to send you an encrypted file, you need to give them your
public key. Use a "secure" channel transfer it to them.

Now, the other person (who also has **LibreSSL** or **OpenSSL** installed) can use
your public key to encrypt a file and send it to you:
```bash
$ openssl rsautl -encrypt -in secret.txt -pubin -inkey pubkey.pem -out secret.txt.enc
```

> If you're encrypting and decrypting on the same machine for testing purposes
make sure that you ([naively](https://ssd.eff.org/en/module/how-delete-your-data-securely-linux)) delete the original file: `rm secret.txt`

After you receive the encrypted file, use your encrypted private key to decrypt
it:
```bash
$ openssl rsautl -decrypt -in secret.txt.enc -out secret.txt -inkey privkey.pem.enc
Enter pass phrase for privkey.pem.enc:
```

Enter the password for decrypting your private key and you're done!

```bash
$ cat secret.txt
USER: dusan
PASS: password123
```

## Verdict:

**Symmetric key use case:**:
```
Security : 5.5
UX       : 8
```

**Aymmetric key use case:**:
```
Security : 9
UX       : 7
```

# 4. GnuPG

As the `gpg` man page says:
> `GnuPG` is a tool to provide digital encryption and signing services using the
OpenPGP standard. `GnuPG` features complete key management and all the bells and
whistles you would expect from a full OpenPGP implementation.

In short, `GnuPG` is a tool for secure communication, be it email encryption or
secure file sharing as in the case of this article.

## Symmetric key use case

You can use `GnuPG` for **symmetric key** encryption, but I'm not going
to cover it in detail. Here is just a short example:

```bash
# Run the encryption command and enter the symmetric key passphrase twice:
$ gpg -c secret.txt
$ rm secret.txt
$ ls
secret.txt.gpg

# Send the encrypted file to the other computer and decrypt it using the same
# passphrase:
$ gpg -d -o secret.txt secret.txt.gpg
gpg: AES encrypted data
gpg: encrypted with 1 passphrase
$ cat secret.txt
USER: dusan
PASS: password123
```

Now let's get to the more secure, **asymmetric key** use case...

## Asymmetric key use case

### Generating a GnuPG private/public key pair

I advise that you use the `gpg --full-gen-key` command for key generation,
since it's going to offer you the most control. Running this command will
prompt you for a couple of input parameters, first of them being the encryption
algorithm. I will choose `RSA AND RSA`:

```bash
$ gpg --full-gen-key
gpg (GnuPG) 2.2.7; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
 (1) RSA and RSA (default)
 (2) DSA and Elgamal
 (3) DSA (sign only)
 (4) RSA (sign only)
Your selection? 1
```

Next you need to choose the keysize, I will pick `4096 bits` which will make
it opaque to brute-force attacks:
```
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
```

For key validity, I suggest using a definite expiry period. For the sake of an
example I'll choose `3 months`:
```
Please specify how long the key should be valid.
       0 = key does not expire
    <n>  = key expires in n days
    <n>w = key expires in n weeks
    <n>m = key expires in n months
    <n>y = key expires in n years
Key is valid for? (0) 3m
Key expires at Mon Jun 18 22:07:00 2019 CEST
Is this correct? (y/N) y
```

Next up, GnuPG is going to ask for some personal info in order to create a user
ID to go along with the newly created key pair:

1. **Real name:** You can put in whatever you want, but [be sure to use only ASCII
characters](#gnupg-non-ascii-user-identifier-troubleshooting) because otherwise you might introduce bugs.
2. **Email address:** Self explanatory.
3. **Comment:** Whatever you want, I just leave this blank.

```
GnuPG needs to construct a user ID to identify your key.

Real name: Dusan Dimitric
Email address: dusan_dimitric@yahoo.com
Comment:
You selected this USER-ID:
  "Dusan Dimitric <dusan_dimitric@yahoo.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
```

Next you're going to be presented with 2 dialogs where you will choose and
confirm a passphrase that will protect your private key:
```
┌──────────────────────────────────────────────────────┐
│ Please enter the passphrase to                       │
│ protect your new key                                 │
│                                                      │
│ Passphrase: ***********_____________________________ │
│                                                      │
│       <OK>                              <Cancel>     │
└──────────────────────────────────────────────────────┘
```

```
┌──────────────────────────────────────────────────────┐
│ Please re-enter this passphrase                      │
│                                                      │
│ Passphrase: ***********_____________________________ │
│                                                      │
│       <OK>                              <Cancel>     │
└──────────────────────────────────────────────────────┘
```

The GnuPG manual states:

>From the perspective of security, the passphrase to unlock the private key is
one of the weakest points in GnuPG (and other public-key encryption systems as
well) since it is the only protection you have if another individual gets your
private key.

So choose a good passphrase! And it's a **passphrase** not a **password** so
you can use whitespace to string multiple words together if you want.

Now to finish generating the keys - generate some precious entropy (as
instructed) and you're done:
```
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

gpg: key 08BDFF7DC22C37D1 marked as ultimately trusted
gpg: revocation certificate stored as '/home/dusan/.gnupg/openpgp-revocs.d/CF5D0CCA5563381584AD977A08BDFF7DC22C37D1.rev'
public and secret key created and signed.

pub   rsa4096 2019-03-20 [SC] [expires: 2019-06-18]
      CF5D0CCA5563381584AD977A08BDFF7DC22C37D1
uid                      Dusan Dimitric <dusan_dimitric@yahoo.com>
sub   rsa4096 2019-03-20 [E] [expires: 2019-06-18]
```

You can now see your newly generated key pair in the keyring:
```bash
$ gpg --list-keys
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
gpg: next trustdb check due at 2019-06-18
/home/dusan/.gnupg/pubring.kbx
pub   rsa4096 2019-03-20 [SC] [expires: 2019-06-18]
      CF5D0CCA5563381584AD977A08BDFF7DC22C37D1
uid           [ultimate] Dusan Dimitric <dusan_dimitric@yahoo.com>
sub   rsa4096 2019-03-20 [E] [expires: 2019-06-18]
```

Notice that GnuPG takes care of securely storing your private keys so you don't
have to do it manually.

If you want to change some of the key pair parameters just delete the current
key and generate a new one. You can delete a key pair with:
```
gpg --delete-secret-and-public-key dusan_dimitric@yahoo.com
```

### Encryption/Decryption

In order for someone to securely send you a GPG encrypted file you must send
them your public key. In order to send your public key to someone, you must
export it first:

```bash
$ gpg --output my_pubkey.gpg --export dusan_dimitric@yahoo.com
$ ls
my_pubkey.gpg
```

The key we exported is in binary format by default. The `--armor` flag can be
used to export the public key in an `ASCII-armored` format, which makes it easy
to share your public key using email, your webpage or any other textual medium.

Let's remove the binary public key and create an ASCII-armored one:
```bash
$ rm my_pubkey.gpg
$ gpg --armor --output my_pubkey.asc --export dusan_dimitric@yahoo.com
$ ls
my_pubkey.asc
```

>I like to use the `.asc` extension for ASCII-armored keys, and `.gpg` for plain,
binary keys.

In order for someone to securely send you a file with the help of `GnuPG`, they
first have to import your private key and then use it to encrypt the intended
file. You can share your public key with the sender by any means that you want.
Your public key is safe for anyone to see and use.

```bash
$ gpg --import my_pubkey.asc
```

Before using the imported public key, make sure to validate it first.  A key is
validated by verifying the key's fingerprint and then signing the key to certify
it as a valid key. 

```bash
$ gpg --fingerprint dusan_dimitric@yahoo.com
pub   rsa4096 2019-03-24 [SC] [expires: 2019-06-22]
      0C83 BB42 8D9B C164 9182  F5C6 9F2F 6AA3 E63E D6A0
uid           [ultimate] Dusan Dimitric <dusan_dimitric@yahoo.com>
sub   rsa4096 2019-03-24 [E] [expires: 2019-06-22]
```

The fingerprint here is: `0C83 BB42 8D9B C164 9182  F5C6 9F2F 6AA3 E63E D6A0`.

You must verify that the fingerprint is the same by communicating with the
public key's owner. This can be in person, over the phone or any other secure
method. If those 20 bytes match, you can go ahead and proceed with encryption
using this verified public key.

```bash
$ gpg --encrypt --output secret.txt.enc --recipient dusan_dimitric@yahoo.com secret.txt
$ ls
my_pubkey.gpg  secret.txt  secret.txt.enc
```

The sender will send the encrypted file to you, using whichever channel they
want. When you get the encrypted file, you can decrypt it using:
```bash
$ gpg --output secret.txt --decrypt secret.txt.enc
```

You will be prompted to enter the private key passphrase, and if the correct
one is entered you will have your decrypted file:
```bash
$ cat secret.txt
USER: dusan
PASS: password123
```

## Verdict:
```
Security : 9+
UX       : 7
```

# Troubleshooting: GnuPG Non-ASCII User Identifier:

Let's say you create a GnuPG key pair and use non-ASCII characters in the `Name`
field (I used `Dušan Dimitrić`, with `š` and `ć` obviously being non-ASCII).
Doing that will cause the remove keys command not to work, along with other
unexpected bugs:

```bash
$ gpg --list-keys
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
gpg: next trustdb check due at 2019-06-19
/home/dusan/.gnupg/pubring.kbx
--------------------------------------
pub   rsa4096 2019-03-21 [SC] [expires: 2019-06-19]
      C5875F42BB3439D2505E032D1E93F87508BC08D0
uid           [ultimate] Dušan Dimitri\xc4\x20<dusan_dimitric@yahoo.com>
sub   rsa4096 2019-03-21 [E] [expires: 2019-06-19]
```

Trying to remove the key pair will fail, no matter if you use the `Name`,
`Email` or fingerprint to identify and delete the key:
```bash
$ gpg --delete-secret-and-public-key Dušan Dimitri\xc4\x20
gpg (GnuPG) 2.2.7; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.


sec  rsa4096/1E93F87508BC08D0 2019-03-21 Dušan Dimitri\xc4\x20<dusan_dimitric@yahoo.com>

Delete this key from the keyring? (y/N) y
This is a secret key! - really delete? (y/N) y
gpg: deleting secret key failed: A locale function failed
gpg: deleting secret subkey failed: A locale function failed
gpg: Dušan: delete key failed: A locale function failed
```

Easiest way to solve this is to remove all surface level files from your
`~/.gnupg` directory:
```bash
cd ~/.gnupg
rm *
```

Now everything regarding keys and keyrings is deleted and `GnuPG` will
re-create the missing files next time you run it.

**Lesson learned: Don't use non-ASCII characters for any of your GnuPG key
identifiers.**

# Bonus 1: GnuPG Key Management

The most cumbersome part of using GnuPG so far is the public key distribution.
It simply does not scale well. The straight-forward solution to sharing your
public key to everyone interested would be to host it on one of the GnuPG
`keyservers`.

I like `keys.gnupg.net` because of it seems popular enough and has a convenient
web interface: [http://keys.gnupg.net](http://keys.gnupg.net)

> At the time of writing this, `http://keys.gnupg.net` seems like one of the
few keyservers that actually work. If someone knows a reliable keyserver shoot
me an email.

Consult the manual for more information regarding [key distribution](https://www.gnupg.org/gph/en/manual.html#AEN464).

# Bonus 2: GnuPG Signatures

In order to make sure that an encrypted file is coming from the sender that
you are expecting it from, a digital signature can be employed.

The sender will _digitally sign_ the file with their **private key** and you,
the receiver, will verify that signature using the sender's **public key**.

Sender:
```bash
$ gpg --output secret.txt.sig --detach-sig secret.txt.enc
```

Receiver:
```bash
$ gpg --verify secret.txt.sig secret.txt.enc
gpg: Signature made Thu Apr  4 19:33:31 2019 CEST
gpg:                using RSA key 0C83BB428D9BC1649182F5C69F2F6AA3E63ED6A0
gpg: Good signature from "Dusan Dimitric <dusan_dimitric@yahoo.com>" [ultimate]
```

If the verification passes - you can be sure that the file is sent by the owner
of the **public key** that you used for the verification. Either that, or their
private key has been compromised.

You can also use signatures for regular, non-encrypted files to ensure that
they haven't been tampered with.

# Bonus 3: GnuPG Public Key Verification & _Web of Trust_

To prevent man-in-the-middle attacks, it would be wise to develop your `web of
trust` - a network of people whose public keys you can use with a certain
degree of trust.

A `web of trust` is formed by people signing, and therefore validating each
other's public keys, making them more trustworthy and tamper-resistant.

The process goes like this:
1. Check the fingerprint of the other person's public key
2. Sign the public key if the fingerprint is correct
3. Check to see if your newly added signature is there
4. Send them the signed public key so they can import it in their keychain and upload to a keyserver
5. Or - send the signed public key to a public keyserver and the owner of the public key can pull it down along with all the new signatures

You can organize key signing within your company, or, if you are using GnuPG
privately you can develop this `web of trust` with your friends, people you
work with and generally people you trust.

If you want to learn more about signing and verification make sure to consult the [GnuPG manual](https://www.gnupg.org/gph/en/manual.html#REVOCATION).

Pay attention to [Importing a public key](https://www.gnupg.org/gph/en/manual.html#AEN84),
[Validating other keys on your public keyring](https://www.gnupg.org/gph/en/manual.html#AEN335)
and [Distributing keys](https://www.gnupg.org/gph/en/manual.html#AEN464) sections of the manual.

# Pushing Privacy Further - Final Tips

I will leave you with some of my favorite ways to increase security to the
**MAX**!

## Tip #1 - Encrypt Your Storage Medium

Encrypting your storage partition will add another layer of protection. In the
case that someone grabs a hold of your hardware they will have a difficult time
accessing any of your files including your private keys.

## Tip #2 - Use Secure Channels of Communication

For exchanging the **encrypted files** and **encryption/decryption
passwords** make sure to use a secure service - something that has E2E
encryption... or at least it markets itself that way.

Here are some of the unsecure channels:

* Email
* SMS
* Cellular calls
* Proprietary services

## Tip #3 - Expire Encrypted Files

Upload the encrypted files somewhere where you can expire them - have the
service delete the files after a short period of time.

## Tip #4 - Tunnel traffic through VPN

This could prevent any eavesdroppers, including your ISP. Choosing a VPN that
you can trust is not an easy job though.

## Tip #5 - Use Layers of Encryption

Combine encryption tools to enforce practically impenetrable communication.
By combining multiple tools you mitigate the risk of one of them being
compromised or zero-day exploited.

Here are some ideas:

## Scenario 1:

**Encryption:**
1. Encrypt the file using **ccrypt** - with an **encryption password**
2. Encrypt the output of the previous step with **LibreSSL** or **GnuPG**, using the recipients **public key**
3. Safely remove the output from step #1
4. Send the encrypted file using a "secure" communication channel - **channel 1**
5. Send **encryption password** using a "secure" **channel 2**

**Decryption:**
1. Get the file via **channel 1**
2. Get the **encryption password** via **channel 2**
3. Decrypt using your **LibreSSL** or **GnuPG** private key
4. Decrypt using **ccrypt** and **encryption password**

## Scenario 2:

**Encryption:**
1. Encrypt the secret file using **LibreSSL AES-256** and a password
2. Encrypt a file containing the password with **GnuPG** using the recipients **public key**
3. Send the encrypted password file and the encrypted secret file to the recipient via any 2 "secure" channels

**Decryption:**
0. Retreive the **password** and **secret** files
1. Decrypt the password file using your **GnuPG** private key
2. Decrypt the secret file using **LibreSSL AES-256** and the decrypted password

Etc. etc... you can combine any of the tools  however you want.

If you wish to add some more tips or have any comments regarding this post or
any other - shoot me an email at: **dusan_dimitric@yahoo.com** (GPG encrypted,
of course :) ).
