ssh-gpg-smartcard-config for yubikey-neo
========================================

This document covers the procedure for configurating a YubiKey as a GPG smartcard for SSH authentication, it also covers setting the correct serial number on the card. The benefit is a good model for `two-factor authentication <http://en.wikipedia.org/wiki/Two-factor_authentication>`_, something you have and something you know. In this example, there is a token and a passphrase.

The `YubiKey Neo <https://www.yubico.com/products/yubikey-hardware/yubikey-neo>`_ is used here. Other yubikeys will not work, as they do not support the applet functionality.

Examples below are using a Fedora 22 x86_64 and Ubuntu 15.04 x86_64 fresh install. There are other tutorials for other operating systems and keys available online. See the CREDITS section below for alternate tutorials, examples, etc.

Configuring Authentication with GNOME-Shell
-------------------------------------------
To configure authentication using the previously generated GnuPG key, the GNOME-Shell needs some adjustements. With help from several resources, configure the system to allow ``gpg-agent`` to take over SSH authentication.

Certain software must be installed, including utilities for the YubiKey ``libyubikey-devel`` (``libyubikey-dev`` on Ubuntu), ``gnupg2`` (which is probably already installed), ``gnupg2-smime`` (``gpgsm`` on Ubuntu)i, ``pcsc-lite-ccid``, and ``pcsc-lite`` (``pcscd`` and ``libpcsclite1`` on Ubuntu).

*Fedora*::

  $ sudo dnf install ykpers-devel libyubikey-devel libusb-devel \
     autoconf gnupg gnupg2-smime pcsc-lite pcsc-lite-ccid

*Ubuntu*::

  $ sudo apt-get install gnupg-agent gnupg2 pinentry-gtk2 scdaemon \
     libccid pcscd libpcsclite1 gpgsm yubikey-personalization \
     libyubikey-dev libykpers-1-dev

**Optional**: Install the `Yubikey NEO Manager GUI <https://developers.yubico.com/yubikey-neo-manager/>`_. If running Ubuntu, you can install the yubikey neo manager and other yubikey software from the `Yubico PPA <https://launchpad.net/~yubico/+archive/ubuntu/stable>`_.

Enable your YubiKey NEO’s Smartcard interface (CCID)
-----------------------------------------------------
This will enable the smartcard portion of your yubi key neo::

  $ ykpersonalize -m82

If you have a dev key, Reboot your yubikey (remove and reinsert) so that ykneomgr works.

Configure GNOME-Shell to use gpg-agent and disable ssh-agent
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configure GPG to use its agent (only for smartcard)::

  $ echo "use-agent" >> ~/.gnupg/gpg.conf

Enable ssh-agent drop in replacement support for gpg-agent::

  $ echo "enable-ssh-support" >> ~/.gnupg/gpg-agent.conf

Allow admin actions on your YubiKey (if your gnupg version is < 2.0.11)::

  $ echo "allow-admin" >>  ~/.gnupg/scdaemon.conf


Intercept gnome-keyring-daemon and put gpg-agent in place for ssh authentication (Ubuntu)
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''
Open Startup Applications

Uncheck "GPG Password Agent" and "SSH Key Agent"

.. image:: startup_apps_checked.png

Edit ``/usr/share/upstart/sessions/gpg-agent.conf`` so that the pre-start script contains the following::

  eval "$(gpg-agent --daemon --enable-ssh-support --sh)" >/dev/null
  initctl set-env --global GPG_AGENT_INFO=$GPG_AGENT_INFO
  initctl set-env --global SSH_AUTH_SOCK=$SSH_AUTH_SOCK
  initctl set-env --global SSH_AGENT_PID=$SSH_AGENT_PID

Add the following lines to the post-stop script section::

  initctl unset-env --global SSH_AUTH_SOCK
  initctl unset-env --global SSH_AGENT_PID

Comment out the ``use-ssh-agent`` line in ``/etc/X11/XSession.options`` file.

Disable the other system gpg-agent::

  mv /etc/X11/Xsession.d/90gpg-agent ~/bak/90gpg-agent

Note: We could have used the Xsession gpg-agent and trashed the upstart one, but there is an `open bug report <https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=642021>`_ for 90gpg-agent. Also, the upstart script has the capability of exporting the environment variables globally with initctl set-env --global.

Intercept gnome-keyring-daemon and put gpg-agent in place for ssh authentication (Fedora)
''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''
If running gnome, you have to prevent gnome-keyring from autostarting its ssh-agent implementation, because we will use GPG's instead:

  sed -e '$aX-GNOME-Autostart-enabled=false' -e '$aHidden=true' \
      /etc/xdg/autostart/gnome-keyring-ssh.desktop > $HOME/.config/autostart/gnome-keyring-ssh.desktop

Now create a new .desktop file to autostart GPG agent.  If you are running in GNOME 3.16 or earlier call the file gnome-keyring-gpg.desktop instead.  It will then override the broken gpg-agent in GNOME 3.16 and earlier, which does not support smartcards.

  wget -O ~/.gnupg/gpg-agent-gnome https://github.com/lfit/ssh-gpg-smartcard-config/raw/master/gpg-agent-gnome && chmod +x ~/.gnupg/gpg-agent-gnome
  cat > $HOME/.config/autostart/gpg-agent-gnome.desktop << EOF
  [Desktop Entry]
  Type=Application
  Name=GPG Password Agent
  Comment=GNOME Keyring: GPG Agent
  Exec=$HOME/.gnupg/gpg-agent-gnome
  OnlyShowIn=GNOME;Unity;MATE;
  X-GNOME-Autostart-Phase=Initialization
  X-GNOME-AutoRestart=false
  X-GNOME-Autostart-Notify=true
  EOF

Reload GNOME-Shell So that the gpg-agent stuff above takes effect. 
------------------

Rebooting the machine works the best.


Get gpshell etc to fix serial number*
--------------------------------
#\* This section not relevant to a consumer edition NEO, it can still be relevant to a developer edition NEO. This section has not been tested with Ubuntu.

Install gpshell binary and libs from tykeal's repo::

  $ sudo yum install http://copr-be.cloud.fedoraproject.org/results/tykeal/GlobalPlatform/fedora-19-x86_64/tykeal-GlobalPlatform-release-0.0.1-1.fc19/tykeal-GlobalPlatform-release-0.0.1-1.fc19.x86_64.rpm

  sudo yum install gpshell gppcscconnectionplugin


Create a gpinstall file::

  cat <<EOF >> gpinstall.txt
  mode_211
  enable_trace
  establish_context
  card_connect
  select -AID a000000003000000
  open_sc -security 1 -keyind 0 -keyver 0 -mac_key 404142434445464748494a4b4c4d4e4f -enc_key 404142434445464748494a4b4c4d4e4f
  delete -AID D2760001240102000000000000010000
  delete -AID D27600012401
  install -file openpgpcard.cap -instParam 00 -priv 00
  card_disconnect
  release_context
  EOF


Get the cap file and place it where gpinstall expects to find it::

  wget -O openpgpcard.cap https://github.com/Yubico/yubico.github.com/raw/master/ykneo-openpgp/releases/ykneo-openpgp-1.0.5.cap



put the correct serial number into gpinstall.txt:: 

  if ykneomgr -s; then
    sed -i "s/^install.*/& -instAID D276000124010200006"$(printf %08d "$(ykneomgr -s)")"0000/" gpinstall.txt
  fi


Flash the card\*::

  gpshell gpinstall.txt

#\* WARNING This erases all existing keys on the smartcard

#\* End section not relevant to a consumer edition NEO

Setting PINs
------------

Included with the gemalto token and GnuPG Smartcard version 2 should be a document describing the default PIN values. There is a regular PIN, which is used to unlock the token for Signing, Encryption or Authentication. Additionally, there is an admin PIN, which is used to reset the PIN and/or the Reset Code for the key itself.


Complete these steps for PIN and then Admin Pin
~~~~~~~~~~~~~~~~~
default pins are 123456 and 12345678 respectivly 

::

  $ gpg2 --card-edit
  ..snip..

  gpg/card> admin
  Admin commands are allowed

  gpg/card> passwd
  gpg: OpenPGP card no. D27600012401020000050000158A0000 detected

  1 - change PIN
  2 - unblock PIN
  3 - change Admin PIN
  4 - set the Reset Code
  Q - quit

  Your selection? 3

Enter the Current Admin PIN

.. image:: http://sexysexypenguins.com/misc/gpg-admin.png

Then enter the New Admin PIN twice

.. image:: http://sexysexypenguins.com/misc/gpg-new-admin.png

PIN
~~~

::

  1 - change PIN
  2 - unblock PIN
  3 - change Admin PIN
  4 - set the Reset Code
  Q - quit

  Your selection? 1

Enter the Current PIN

.. image:: http://sexysexypenguins.com/misc/gpg-pin.png

Then enter the New PIN twice

.. image:: http://sexysexypenguins.com/misc/gpg-new-pin.png

**NOTE:** If the Admin PIN has not been entered, it may be required before changes are applied.

Generating an SSH Key using GnuPG
---------------------------------

There are several ways to generate an SSH Key using GnuPG. A common way is to link the new authentication key to an already existing key::

  $ gpg2 --edit-key 8A8F1D53
  gpg (GnuPG) 1.4.12; Copyright (C) 2012 Free Software Foundation, Inc.
  This is free software: you are free to change and redistribute it.
  There is NO WARRANTY, to the extent permitted by law.

  Secret key is available.

  pub  3072R/8A8F1D53  created: 2012-10-06  expires: never       usage: SC
                     trust: ultimate      validity: ultimate
  sub  3072R/2F15E06B  created: 2012-11-23  expires: 2022-11-21  usage: S
  sub  3072R/EB8B4EBD  created: 2012-11-24  expires: 2022-11-22  usage: E
  sub  3072R/6BB325E9  created: 2012-11-24  expires: 2022-11-22  usage: A
  [ultimate] (1). Clint Savage <herlo1@gmail.com>
  [ultimate] (2)  Clint Savage <herlo@fedoraproject.org>
  [ultimate] (3)  Clint Savage <csavage@linuxfoundation.org>

  gpg>

Once in the ``edit-key`` dialog, create a key on the card::

  gpg> addcardkey
  Signature key ....: 91BC 60CC B9EC 8E73 923A  FC6D 58CD 88A6 2F15 E06B
  Encryption key....: 0CC3 DC3E 0D17 6111 A62B  F656 63C6 4DA9 EB8B 4EBD
  Authentication key: 9EBF A9FE 8AE1 0FEB 1699  CE9A 779F 43D5 EC6F CC13

  Please select the type of key to generate:
     (1) Signature key
     (2) Encryption key
     (3) Authentication key
  Your selection? 3
  
  IT WILL PROMPT YOU TO ENTER THE ADMIN PIN, AND THEN THE REGULAR PIN. Don't fat finger this part!

  gpg: WARNING: such a key has already been stored on the card!

  Replace existing key? (y/N) y
  What keysize do you want for the Authentication key? (3072)
  Key is protected.

  You need a passphrase to unlock the secret key for
  user: "Clint Savage <herlo1@gmail.com>"
  3072-bit RSA key, ID 8A8F1D53, created 2012-10-06

  Please specify how long the key should be valid.
           0 = key does not expire
        <n>  = key expires in n days
        <n>w = key expires in n weeks
        <n>m = key expires in n months
        <n>y = key expires in n years
  Key is valid for? (0) 10y
  Key expires at Mon 21 Nov 2022 05:29:00 PM MST
  Is this correct? (y/N) y
  Really create? (y/N) y
  gpg: Note that the key does not use the suggested creation date

  pub  3072R/8A8F1D53  created: 2012-10-06  expires: never       usage: SC
                       trust: ultimate      validity: ultimate
  sub  3072R/2F15E06B  created: 2012-11-23  expires: 2022-11-21  usage: S
  sub  3072R/EB8B4EBD  created: 2012-11-24  expires: 2022-11-22  usage: E
  sub  3072R/6BB325E9  created: 2012-11-24  expires: 2022-11-22  usage: A

  [ultimate] (1). Clint Savage <herlo1@gmail.com>
  [ultimate] (2)  Clint Savage <herlo@fedoraproject.org>
  [ultimate] (3)  Clint Savage <csavage@linuxfoundation.org>

Upon completion of the key, be sure to save the record to the card and gpg key::

  gpg> save
  $

Verify SSH key is managed via gpg-agent
---------------------------------------

Assuming everything above is configured correctly, a simple test is performed with the SmartCard inserted::

  $ ssh-add -L
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDL/XmU......BL0luE= cardno:00050000158A

FILES
-----

`The github repository <https://github.com/herlo/ssh-gpg-smartcard-config/>`_ contains all the files to make the changes above. Please feel free to read through them.

CREDITS
-------

A special thanks to the following people and/or links.

  * `How to use GPG with SSH (with smartcard section) <http://www.programmierecke.net/howto/gpg-ssh.html>`_
  * `The GnuPG Smartcard HOWTO (Advanced Features) <http://www.gnupg.org/howtos/card-howto/en/smartcard-howto-single.html#id2507402>`_
  * `Smart Cards and Secret Agents <http://blog.flameeyes.eu/2010/08/smart-cards-and-secret-agents>`_
  * `How to mitigate issues between gnupg and gnome keyring manager <http://wiki.gnupg.org/GnomeKeyring>`_
  * `Useful info on how to start the correct agent at login <http://www.bootc.net/archives/2013/06/09/my-perfect-gnupg-ssh-agent-setup/>`_
