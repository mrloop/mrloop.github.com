---
layout: post
title: "Pinentry"
comments: true
categories: [workflow]
---

# Keys #

I regulary use gpg and ssh keys during the day as part of my workflow, from signing git commits, encrypting sensetive messages to email and authenticating with servers. I regularly use gpg via Enginmail and the Thunderbird GUI and from the command line. Most of my of ssh keys is from the command line. I was having a couple of issues with the default ubuntu 16.04 setup for managing keys.
1. gnome-keyring-agent was not always handling encrypted private keys correctly resulting in some scripts failing to authticate with an error message `too many authentication attempts` even though `ssh-add -l` showed the correct keys where available.
2. using `ssh`, `ssh-add` from `gpg2` from the command line when a password was requested to descrypt a key a GUI dialog would pop up disrupting my focus.

Ideally a terminal based password entry would present it's self when working in a terminal and a GUI dialog present itself when working with a desktop app. A solution would also have to fix the `too many authentication attempts` error.



