# Generate deterministic PGP keys and transfer subkeys to Gnuk token

PGP key generation and transfer of the private subkeys to a smartcard needs to be done on an air-gapped machine.  Furthermore, even when generated on the air-gapped machine, the keys must never touch any disk based filesystem or get cached to any disk based swap.  The private keys must only reside temporarily in a tmpfs (RAM) filesystem before they are transferred to the smartcard and removed from the air-gapped machine tmpfs filesystem.

Follow [Air gapped raspberry pi](<air-gapped-raspberry-pi.md>) instructions.  Once your air-gapped raspberry pi is running, continue with the following instructions.


## Do all the following on an air-gapped machine

Note:  You will need to type every command in manually at the keyboard.  You will not be able to cut and paste anything into the terminal because the raspberry pi is air-gapped.  You must not circumvent the air-gap, so please don't connect any network cables or try to configure wifi.

All required software should already reside on the air-gapped machine.  Power up this machine.


## 1. Confirm that /home directory resides on tmpfs filesystem and there is no swap
```
df
mount
/usr/sbin/swapon --show
```
Ensure these commands show tmpfs mounted at /home and no disk based swap.


## 2. Set correct date and check
This is necessary every time because the raspberry pi does not have a hardware clock.
```
sudo date --set '2024-09-23 22:49:00'

date
```
> Mon 23 Sep 2024 22:49:03 CEST  

If the timezone is incorrect then you can set this by simply symlinking to the correct zone.  Eg: ```sudo ln -sf /usr/share/zoneinfo/<country>/<city> /etc/localtime```


## 3. Generate PGP secret keys
```
cd $HOME

generate_derived_key \
  --key-type eddsa \
  --name 'First Last' \
  --email 'name@domain.com' \
  --key-creation '2020-01-01 00:00:00' \
  --sigtime '2020-01-01 00:00:00' \
  --sigexpiry '2030-01-01 00:00:00' \
  --output-file name@domain.com-secret.key
```
Notes:

- To recreate the same key all the following values need to be identical:
  - --name
  - --email
  - --key-creation
  - Recovery seed
- Recommend setting --sigtime to the same value as --key-creation
- The secret key is saved to $HOME directory which resides on a tmpfs (RAM) filesystem.  Thus the secret key will vanish when the air-gapped machine is powered off.

**It is very important that the secret key is never saved on any disk anywhere because it will be possible for an attacker to recover parts of the key from SSD drives.  The air-gapped machine has been configured with no swap to remove the possibility that the OS might cache the key to disk based swap.**


## 4. Import generated secret keys, delete the secret key file, and show all keyring keys
```
gpg --import name@domain.com-secret.key

rm name@domain.com-secret.key

gpg --list-secret-keys

gpg --list-keys
```
> sec   ed25519 2024-09-23 [C] [expires: 2024-09-24]  
>       blahblahblahblahblahblahblahblahblahbla  
> uid           [ unknown] User Name <user@domain.com>  
> ssb   ed25519 2024-09-23 [SC] [expires: 2024-09-24]  
> ssb   cv25519 2024-09-23 [E] [expires: 2024-09-24]  
> ssb   ed25519 2024-09-23 [A] [expires: 2024-09-24]  


## 5. Export the public key and display as a QR Code on the screen
```
gpg --export --armor name@domain.com | qrencode --margin 2 --type utf8
```
Use your mobile camera to scan the image and your email app to send the public key information to your localPC.

### 5.1 Alternatively, export the public key to root's directory (on permanent storage)
```
gpg --export --armor name@domain.com | sudo tee /root/name@domain.com-public.key >/dev/null
```
Note:

- Public key saved to root's home directory since pi's home directory is on tmpfs.
- To copy the public key: Power off the air-gapped machine, remove the SD card, and insert it in your local PC.


## 6 Insert USB Gnuk token and check it is recognised
```
gpg --card-status
```
> Reader ...........: 234B:0000:FSIJ-2.2-3931CF92:0  
> Application ID ...: D276000124010200FFFE3931CF920000  
> Application type .: OpenPGP  
> Version ..........: 2.0  
> Manufacturer .....: unmanaged S/N range  
> Serial number ....: 3931CF92  
> Name of cardholder: [not set]  
> Language prefs ...: [not set]  
> Salutation .......:  
> URL of public key : [not set]  
> Login data .......: [not set]  
> Signature PIN ....: forced  
> Key attributes ...: ed25519 cv25519 ed25519  
> Max. PIN lengths .: 127 127 127  
> PIN retry counter : 3 3 3  
> Signature counter : 0  
> KDF setting ......: off  
> Signature key ....: [none]  
> Encryption key....: [none]  
> Authentication key: [none]  
> General key info..: [none]  

Note: The Gnuk token must not already have keys on it, or if it does then you will need to factory-reset it: ```gpg --card-edit``` then ```gpg/card> admin``` to access admin commands, then ```gpg/card> factory-reset```.

If you only have a used Gnuk token without the factory-reset option, then you will need to manually reprogram it.  See: [Reprogram fst-01 with Gnuk](<reprogram-fst-01-with-gnuk.md>)


## 7. Initial Gnuk token configuration

Gnuk specific things:

- Gnuk doesn't allow setting passphrase before importing private keys.  Only _after_ importing the private keys is it possible to change the passphrase.
- Thus setup of Gnuk needs to occur in the following order:
  - Initial configuration
  - Import private keys
  - Set passphrase
- Gnuk supports “admin-less mode” for your passphrase setting. It’s the smartcard culture to have two passphrases (one for admin, another for user). Gnuk supports the use case where admin==user.

### 7.1 Setup KDF-DO
KDF-DO is a feature of OpenPGP card to allow computation of key derivation function on host side and is mandatory for Gnuk 2.2. KDF-DO enables the private keys on Gnuk's MCU flash ROM to be encrypted securely with help from the host machine.
```
gpg --card-edit

gpg/card> admin
Admin commands are allowed

gpg/card> kdf-setup single
```
Notes:
- 'kdf-setup' is the sub-command and 'single' specifies the use case of single PIN ("admin-less mode").
- You will need to enter the factory Admin PIN which is '12345678'.

### 7.2 Personalise Gnuk
Optionally configure name, language, salutation, url and login.  Maybe just configure login for now:
```
gpg/card> login
Login data (account name): username
```

### 7.3 Configure to not require PIN input everytime for signing
Toggle the forcesig setting to not force PIN for signature everytime
```
gpg/card> forcesig
```
This will toggle forcesig setting to off.

The forcesig setting of 'not forced' enables the gpg-agent to cache and re-use the passphrase when Gnuk is used the first time following insertion.  The passphrase will be cached until Gnuk is removed from USB.

Note: You can also configure gpg-agent with a line "Confirm: yes" to make gpg request authorisation of key use every time by clicking a desktop notification.

### 7.4 Quit interactive session
End of initial Gnuk token configuration
```
gpg/card> quit
```


## 8. Send secret subkeys to Gnuk token

### 8.1 Edit your master key
Specify the key to edit by keyid or email address.
```
gpg --edit-key user@domain.com
```

### 8.2 Select signing subkey and send it to Gnuk token (subkey No. 1)
```
gpg> key 1
```

Send this subkey to Gnuk
```
gpg> keytocard
```

Select option 1 for the 'Signature key' and enter the password for the key (or not if the key password is empty).

You will then be asked for the Gnuk Admin PIN.  The default Gnuk Admin PIN is '12345678' so enter this. (You will be asked to enter it twice.)


### 8.3 Unselect subkey 1 and select the encryption subkey (2) and send it to Gnuk
```
gpg> key 1
gpg> key 2
gpg> keytocard
```
Password for key.  Password for Gnuk Admin '12345678'

### 8.4 Do the same for the authentication subkey (3)
```
gpg> key 2
gpg> key 3
gpg> keytocard
```
Password for key.  Password for Gnuk Admin '12345678'

Save the gpg changes.
```
gpg> save
```

Now the secret subkeys are on the Gnuk token and have been removed from your local gpg instance.  Ensure that you did delete the saved copy of the private keys in your $HOME/.gnupg directory.


## 9. Set passphrase for Gnuk token
The OpenPGPcard specification has two passwords: user-password and admin-password. The user-password is refered as PW1, and admin-password is refered as PW3.

### 9.1 Edit the Gnuk smartcard
```
$ gpg --card-edit
```
> Reader ...........: 234B:0000:FSIJ-1.2.0-87193059:0  
> Application ID ...: D276000124010200FFFE871930590000  
> Version ..........: 2.0  
> Manufacturer .....: unmanaged S/N range  
> Serial number ....: 87193059  
> Name of cardholder: [not set]  
> Language prefs ...: [not set]  
> Salutation .......:  
> URL of public key : [not set]  
> Login data .......: gniibe  
> Signature PIN ....: not forced  
> Key attributes ...: ed25519 cv25519 ed25519  
> Max. PIN lengths .: 127 127 127  
> PIN retry counter : 3 3 3  
> Signature counter : 0  
> KDF setting ......: single  
> UIF setting ......: Sign=off Decrypt=off Auth=off  
> Signature key ....: 249C B377 1750 745D 5CDD  323C E267 B052 364F 028D  
>       created ....: 2015-08-12 07:10:48  
> Encryption key....: E228 AB42 0F73 3B1D 712D  E50C 850A F040 D619 F240  
>       created ....: 2015-08-12 07:10:48  
> Authentication key: E63F 31E6 F203 20B5 D796  D266 5F91 0521 FAA8 05B1  
>       created ....: 2015-08-12 07:16:14  
> General key info..: pub  ed25519/E267B052364F028D 2015-08-12 NIIBE Yutaka <gniibe@fsij.org>  
> sec>  ed25519/E267B052364F028D  created: 2015-08-12  expires: never  
>                                 card-no: FFFE 87193059  
> ssb>  cv25519/850AF040D619F240  created: 2015-08-12  expires: never  
>                                 card-no: FFFE 87193059  
> ssb>  ed25519/5F910521FAA805B1  created: 2015-08-12  expires: never  
>                                 card-no: FFFE 87193059  


### 9.2 Set user PIN (passphrase)
Gnuk is setup with a user PIN and an admin PIN which can both be different.  However, the software is configured to enter "admin-less" mode if _only_ the user PIN is changed.  Changing _only_ the user PIN will also result in the admin PIN being set to equal the user PIN.  IE: PW1 == PW3.  For this to work, the PIN length needs to be >= 8 characters for "admin less mode".
```
gpg/card> passwd
gpg: OpenPGP card no. D276000124010200FFFE871930590000 detected

Please enter the PIN
Enter PIN: 123456

New PIN
Enter New PIN: <PASSWORD-OF-GNUK>

New PIN
Repeat this PIN: <PASSWORD-OF-GNUK>
PIN changed.
```
Note: You will be asked first for the user PIN.  Enter the factory default of '123456'.

Then enter your new passphrase.  It can be anything you like - all letters, numbers, spaces, and punctuation characters are available.

Then select save or quit to exit from 'card-edit' mode.
```
Your selection? Q
gpg/card> quit
```

## 10. Remove Gnuk from USB and re-insert
This is important because it makes gpg-agent forget the old user password which it has cached.


## 11. Test the Gnuk token
Insert Gnuk into your localPC and display output of ```ssh-add -L``` then check if you can ssh into a server that has your key authorising access.  Or, access your password-store which is encrypted with your key.  Or clone a private repository configured to use your key for authentication.  Or test login to your Amazon or Paypal account which require 2FA using one-time passwords enabled by the password-store OTP "google authenticator" plugin.


## 12. Optionally re-edit Gnuk and do more customising
This can now safely be done on your localPC without any risk of exposing your secret keys.
```
gpg --card-edit

gpg/card> admin
```

Type list to list the settings of the Gnuk key
```
gpg/card> list
```

Quit customisation
```
gpg/card> quit
```


## 13. Optionally upload your public key to the keyservers
This cannot be done from the air-gapped machine since it is... well, air-gapped.

You only need to do this again if you extended the expiry of your key or generated an entirely new key using different configuration settings.

Obtain a copy of the public key from the air-gapped machine by accessing the SD card in your PC and copying it over.  Then import the public key into your localPC gnupg instance: ```gpg --import name@domain.com-public.key```

Now send the public key to the default keyserver:
```
gpg --send-keys 0xblahblah_my_keyid_blahblah
```

Or send it to a specific keyserver:
```
$ gpg --keyserver pool.sks-keyservers.net --send-key 0xblahblah_my_keyid_blahblah
```

Now check that you can download your public key from the keyservers:
```
$ gpg --recv-key 0xblahblah_my_keyid_blahblah
```


## References:
- https://blog.summitto.com/posts/deterministic_pgp_deep_post/
- https://github.com/summitto/pgp-key-generation
- https://www.fsij.org/doc-gnuk/index.html



****
## Tips
****

### Script to generate a random 24 word seed phrase using openssl rand (on the air-gapped machine)
When generating a new PGP key, the generate_derived_key program suggests to randomly generate a 24 word seed phrase by rolling a 6 sided dice 100 times and entering each roll result.  However, depending on your level of paranoia, you might consider the bash script using the openssl rand utility to generate 24 random bip-39 seed words installed at: /usr/local/bin/random-bip39-words




