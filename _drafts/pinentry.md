---
layout: post
title: "Pin entry"
comments: true
categories: [workflow]
tags: [zsh, gpg, ssh, keys]
---

# Private Keys #

I regularly use gpg and ssh keys during the day, from signing git commits, encrypting sensitive messages to email and authenticating with servers. I regularly use gpg via [Enignmail](https://enigmail.net/) and the [Thunderbird](https://www.mozilla.org/en-US/thunderbird/) GUI and from the command line in [zsh](http://zsh.sourceforge.net/). I was having a couple of issues with the default ubuntu 16.04 setup for managing keys.
1. gnome-keyring-agent was not always handling encrypted private keys correctly resulting in some scripts failing to authenticate with an error message `too many authentication attempts` even though `ssh-add -l` showed the correct keys where available.
2. using `ssh`, `ssh-add` from `gpg2` from the command line when a password was requested to decrypt a key a GUI dialog would pop up disrupting my focus.

Ideally a terminal based password entry would present it's self when working in a terminal and a GUI dialog present itself when working with a desktop app. A solution would also have to fix the `too many authentication attempts` error.

#### Solution ####

The [`gpg-agent`](https://www.gnupg.org/documentation/manuals/gnupg/Invoking-GPG_002dAGENT.html#Invoking-GPG_002dAGENT) is a good candidate, it is a daemon to manage secret keys similar the [`ssh-agent`](http://man.openbsd.org/OpenBSD-current/man1/ssh-agent.1) or the [`gnome-keyring-daemon`](https://wiki.gnome.org/Projects/GnomeKeyring/Ssh). It's used by `gpg2` and can be used as an agent for ssh. `gpg-agent` has numerous [pinentry](https://www.gnupg.org/related_software/pinentry/index.en.html) programs that allow gpg to read passphrases in a secure manner, there are GUI and terminal based programs and the `gpg-agent` can be configured to use different pinentry programs.

I updated GnuPG to the latest version using https://gist.github.com/vt0r/a2f8c0bcb1400131ff51

My `.zshrc` contains the following:

````sh
# gpg
#
GPG_TTY=$(tty)
export GPG_TTY
# gpg as ssh-agent
unset SSH_AGENT_PID
if [ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ]; then
  export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
fi

# let pinentry know which console to display in for ssh-agent
# https://www.gnupg.org/documentation/manuals/gnupg/Common-Problems.html
update-start-up-tty(){
  echo UPDATESTARTUPTTY | gpg-connect-agent > /dev/null
}

# let gpg-agent know which pinentry program to use
update-pinentry-app(){
  echo "term" > /tmp/pinentry-app
}

# add zsh hook to exec above function before every command execution
autoload -U add-zsh-hook
zsh-preexec(){
  update-start-up-tty
  update-pinentry-app
}
add-zsh-hook preexec zsh-preexec

# similar to `ssh-add -D` but works with gpg-agent
ssh-delete(){
  grep -o ^[A-Z0-9]* ~/.gnupg/sshcontrol | xargs -I% rm ~/.gnupg/private-keys-v1.d/%.key
  echo '' > ~/.gnupg/sshcontrol
}
````

Next a simple shim for pinentry is needed, when invoked it selects either a terminal pinentry program or a GUI pinentry program based on contents of `/tmp/pinentry-app`

````sh
#!/bin/bash
#
# http://askubuntu.com/questions/858347/disable-gnome-from-asking-passphrase-in-gui-when-using-ssh-and-gpg-from-terminal#858947
#
# Configuration -- adjust these to your liking
PINENTRY_TERMINAL=/usr/bin/pinentry-curses
PINENTRY_X11=/usr/bin/pinentry-x11

# if /tmp/pinentry-app is "x11" then use gui
if [ $(cat /tmp/pinentry-app) = "x11" ]; then
  exec "$PINENTRY_X11" "$@"
else
  exec "$PINENTRY_TERMINAL" "$@"
fi
````

This needs to be linked to pinentry so it is used as our default pinentry program.

````sh
ln -s ~/bin/pinentry /usr/local/bin/pinentry -f
````

The `gpg-agent` needs ssh enabled and then stopped, it'll automatically restart when needed.
````sh
echo 'enable-ssh-support' >> ~/.gnupg/gpg-agent.conf
gpgconf --kill gpg-agent
````

Next the GUI applications using GnuPG need configured, in my case Thunderbird Enigmail.
First another simple shim is created `~/bin/gpg2-thunderbird`, this time for `gpg2`, which tells `gpg-agent` to use the GUI pinentry.

````sh
#!/bin/sh

echo "x11" > /tmp/pinentry-app
exec "/usr/local/bin/gpg2" "$@"
````

Then `Thunderbird > Enigmail > Preferences > Basic > Files and Directories > Override with` is updated with the location of the `gpg2` shim.

# Success! #

I now have my desired private key management setup how I want it. A terminal based pinentry when working in the terminal and a GUI based pinentry when working from a GUI.

#### GUI pin entry ####
<video controls src="/video/enigmail.m4v"></video>

#### Terminal pin entry ####
<asciinema-player font-size="15" loop="true" autoplay="true" src="/video/gpg-agent.json"></asciinema-player>
<script src="/js/asciinema-player.js"></script>

