# ledgerwallet

## Communicate with key

### Arch

Execute
```sh
sudo pacman -S pcsclite pcsc-tools ccid

sudo systemctl start pcscd
sudo systemctl enable pcscd

# you can verify the communication
pcsc_scan
```

### Ubuntu

```sh
sudo apt-get install pcscd pcsc-tools scdaemon

sudo systemctl start pcscd
sudo systemctl enable pcscd

# you can verify the communication
pcsc_scan
```


## Configure gpg

Execute
```sh
echo "reader-port \"Ledger Token [Nano S] (0001) 01 00\"" > ~/.gnupg/scdaemon.conf
echo "enable-pinpad-varlen" >> ~/.gnupg/scdaemon.conf
```

Edit ~/.gnupg/gpg.conf
```
# Avoid information leaked
no-emit-version
no-comments
export-options export-minimal

# Displays the long format of the ID of the keys and their fingerprints
keyid-format 0xlong
with-fingerprint

# Displays the validity of the keys
list-options show-uid-validity
verify-options show-uid-validity

# Limits the algorithms used
personal-cipher-preferences AES256
personal-digest-preferences SHA512
default-preference-list SHA512 SHA384 SHA256 RIPEMD160 AES256 TWOFISH BLOWFISH ZLIB BZIP2 ZIP Uncompressed

cipher-algo AES256
digest-algo SHA512
cert-digest-algo SHA512
compress-algo ZLIB

disable-cipher-algo 3DES
weak-digest SHA1

s2k-cipher-algo AES256
s2k-digest-algo SHA512
s2k-mode 3
s2k-count 65011712
```

## Generate a gpg key

Install the Open PGP application (you have to enable developper mode in ledger live)

### Configure gpg app on ledger

On ledger, go to
```
settings -> reset -> yes
        -> seed mode -> set on
        -> key template -> signature
        -> choose type -> RSA 4096
        -> set template 
```

Execute
```sh
gpgconf --kill all
gpg --edit-card
admin
generate
1y
y
```

if key exists (Key generation failed: File exists), then execute
```sh
gpg-connect-agent "keyinfo --list" /bye
# foreach keygrip linked to your ledger
gpg-connect-agent "delete_key <keygrip>" /bye
# redo action from <Configure gpg app on ledger>
``` 

Execute
```sh
quit
gpg --list-secret-keys --with-keygrip
```

The generated keys are:
- primary sign & cert (RSA 4096) private key on card
- secondary auth (RSA 2048) private key on card
- secondary encrypt (RSA 2048) private key on card

Note the keygrip of the RSA 4096 key

We want this configuration:
- primary cert (RSA 4096) private key on card
- secondary sign (RSA 4096) private key on PC
- secondary encrypt (RSA 4096) private key on PC

So we won't need ledger each time we commit in git, receive or send an encrypted mail, or sign a mail.
Ledger will be use only the certify subkey and change expiration date

So you can delete the created key
```sh
gpg --delete-secret-and-public-keys <fingerprint>
```

### Generate a gpg key from keygrip

An expiration date is usefull even for a primary key, indeed if you lose your 24 words after the expiration gpg users will know which key is currently yours.

Generate the primary key, execute
```sh
gpg --expert --full-generate-key
13
<keygrip>
S
E
Q
1y
y
```

Generate the secondary keys, execute
```sh
gpg --edit-key <fingerprint>
addkey
4
4096
1y
y
y

addkey
6
4096
1y
y
y

save
```

If you generated your key on one PC and you want to use on another
```sh
# save on generator PC
gpg --export-secret-keys <fingerprint> > sec.gpg
# move sec.gpg to new PC, and execute
gpg --import sec.gpg
```

## Restore if you lost your PC

get your public key (it contains fingerprint & keygrip)
```sh
gpg -a --export <fingerprint> > gpg.asc
```
Publish it somewhere (pastebin or your website)

Then import the private key access to your keyring
```sh
gpg --card-edit
url
https://pastebin.com/raw/<PASTEBIN_TOKEN>
fetch
```

Of course your secondary keys private key are lost, you make them expired by changing their expiration time and create new ones.

## Restore if you lost your ledger (or update its firmware)

Execute
```sh
sudo pacman -S git python-pip swig # archlinux
sudo apt-get install git python3-pip libpcsclite-devel libusb-1.0-0-dev swig # ubuntu

pip install --user pyscard
git clone https://github.com/LedgerHQ/ledger-app-openpgp-card.git
cd ledger-app-openpgp-card/pytools

gpgconf --kill all
python3 -m gpgcard.gpgcli --set-template rsa4096:rsa2048:rsa2048 --seed-key --set-serial <serial> --pinpad
```

## renew key expiration

```sh
gpg --edit-key <fingerprint>
expire
1y
key 1
key 2
expire
1y
save

gpg -a --export <fingerprint> > gpg.asc
```

publish `gpg.asc` content to websites and keyrings
