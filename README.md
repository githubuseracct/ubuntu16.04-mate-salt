## Setting up Ubuntu 16.04 with Mate using salt

This document describes _one way_ of handling the personalization of a new OS.
I am using a staging host and [salt](https://github.com/saltstack/salt) because
I want some automated method by which to reproduce all of my preferences and
because I wanted to play around with salt. _Obviously pretty much everything in
this doc is based on my own preferences._

### Prepping the new host

Log into the new host via the GUI or console, whichever you prefer and make sure
it is up to date and has openssh-server installed.

    sudo apt-get update
    sudo apt-get -y upgrade
    sudo apt-get -y install openssh-server
    (reboot if necessary)

### (Optional) Disable sudo passwords on the new host

If you intend to run any commands over salt that require root permissions, you
will need to disable passwords. This is considered too unsafe for some, so do as
you see fit. If you choose this route consider this:

- `visudo` is safer than echoing to a file, but isn't as copy/pastable
- Using your username instead of a group name helps ensure someone else doesn't
  unintentionally get passwordless sudo access.
- Using `/etc/sudoers.d/[your username]` is preferable so that the main
  `/etc/sudoers` file remains unchanged in case your distro needs to update it
  in the future.

This leaves two main choices:

Either:

    sudo visudo -f /etc/sudoers.d/[username]

or:

    sudo bash -c "
      echo '[username] ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/[username]
      chmod 0440 /etc/sudoers.d/[username]"

`sudo` should now be passwordless. You can log out and back in to verify.

### On your staging host

- Check if you have `pip` (type `which pip` to test). The easiest way to install
  it would be by using your distros' method (apt-get, yum, etc.), but feel free
  to use whatever method you prefer.
- Check if you have `virtualenv` (type `which virtualenv` to test) or install it
  using either your distros' package or via `pip install virtualenv`. Again,
  whatever floats your boat.

### Copy your ssh public key to the new host

Even though you pass the _private key_ to the `ssh-copy-id` command, it is
sending the _public key_. Also, if you have an ssh key agent running, you may
not need to specify which private key. You can run `ssh-add -L` to see if you
have an agent running and if it has the desired keys loaded already.

    ssh-copy-id [-i private_key] [host]

`ssh [host]` should now be passwordless.

### Create a virtual environment on your staging host

I will additionally be making the virtualenv directory a git repo for my own
future purposes. Consider this completely optional.

    virtualenv mate-stage
    cd mate-stage
    source bin/activate
    git init .
    echo -e "bin/\ninclude/\nlib/\nlocal/\nshare/" > .gitignore
    echo -e "config/pki/\ncache/\nsalt.log" >> .gitignore
    git config user.name example
    git config user.email example@example.com
    git add .gitignore
    git commit -m "Initial import"

### Install and configure salt on your staging host

    pip install salt-ssh
    mkdir config
    cat > config/roster << 'EOF'
    mate:
      host: [host or IP]
      port: [Only needed if it is not 22]
      user: [your username]
      priv: [path to your private key]
      sudo: True
    EOF

    cat > config/master << 'EOF'
    root_dir: .
    pki_dir: config/pki
    cachedir: cache
    log_file: salt.log
    file_roots:
      base:
        - states
    EOF
    git add config
    git commit -m "Add initial salt config"

### Test the connection

    salt-ssh -c ./config mate test.ping

### Configure salt for additional packages

This will require the optional disabling of sudo passwords from above.

    mkdir states
    cat > states/addpkgs.sls << EOF
    install_additional_pkgs:
      pkg.installed:
        - pkgs:
          - vim
          - git
    EOF
    git add states
    git commit -m "Add additional pkgs"

### Checklist of things to change

- disable IPv6
- set up network manager and hosts file
- add VPN package
- add startup files
- rearrange the view
- Add helper scripts
- add xrdp
