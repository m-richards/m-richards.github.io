---
title: Remote Access with SSH Quickstart
date: 2023-02-06T20:20:25+11:00
draft: false
categories:
  - Devops
  - Teaching
---

This is a post that I hope to be able to point someone to the next time I'm teaching someone how to ssh into a machine.
I'm sure there are better posts out there on this, but I want this to be terse and pragmatic, without a whole lot of
backstory.

So you want to log into a remote server, either a local server computer or on the cloud. Usually, I'm teaching a windows
speaking audience, where I personally use WSL for ssh, but these instructions are also geared to powershell.

1. Enable the ssh agent. `eval \`ssh-agent\`\` on WSL. In powershell, you might need to start the ssh-agent service. In
   a powershell run as administrator:

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

2. Add the ssh key for the server: `ssh-add ~/.ssh/<path_to_ssh_key>`. Note that for personal keys, this would likely be
   `~/.ssh/id_25519` or `~/.ssh/id_rsa`, and AWS keys end in `.pem` usually.
3. If this throws an error about permissions on WSL, run `chmod 400 ~/.ssh/<path_to_ssh_key>`. This error is about
   protecting your key from being read by other local users, but if for example it's your own personal laptop/
   workstation, that's not exactly a likely scenario.
4. Ssh into the machine: `ssh <username>@<hostname>`. For example, for ec2 this might be something like
   `ssh ubuntu@ec2-198-51-100-1.compute-1.amazonaws.com`

## Troubleshooting

- Try repeating the above with `ssh -v` (verbose), make sure that the key you added was scanned and tried
- Check the username / make sure you haven't forgotten it. It's pretty easy to do
  `ssh ec2-198-51-100-1.compute-1.amazonaws.com`, have the username default to my WSL username and see permission denied
  if I'm not paying attention.
- Make sure you have access to the server. That might involve a VPN connection, making sure your IP address has the
  right access in an AWS security group, ...

## SSH configs

Sometimes, it can be nice to be lazy and shortcut on some of this setup - especially if you're logging into a computer
on EC2 which has an IP address in the name. A simple ssh config looks like this (and conventionally lives at
`~/.ssh/config`)

```bash
Host my_ec2_server
    HostName ec2-198-51-100-1.compute-1.amazonaws.com
    User ubuntu
    IdentityFile ~/.ssh/aws/foo.pem
```

This provides an alias, and allows me to type `ssh my_ec2_server` and log in directly, without adding keys manually,
typing out long IP addresses, or forgetting the username is ubuntu.

Ssh configs can contain multiple hosts, and I find to be a super useful convenience. You can also specify
`ForwardAgent yes` as part of a host declaration which is also useful, as it forwards my ssh keys to the remote machine
\- which lets me clone a repository without leaving keys on a remote machine. You can also `Include` ssh configs inside
ssh configs, which can be handy to share a bunch of configs with colleagues in a more tractable way than copy-paste.

## Aside: Remembering the git clone via ssh syntax

I always prefer cloning repositories via SSH, but until recently I could never for the life of me remember the syntax
for it, as it's not as straightforward as cloning via HTTPS. But it was pointed out to me, that cloning via ssh is the
same syntax as connecting via ssh (I guess the ssh bit was the clue, duh).

To clone a repo via ssh, the command looks like this: `git clone git@github.com:geopandas/geopandas.git`, which if you
think about it, is analogous to SSHing into a computer with

- username `git`
- ip `github.com`
- File located at `geopandas/geopandas.git`

Now I'm certainly not suggesting that all of github is just a really bit computer we're all SCPing our code down from,
but I think it's still a useful mnemonic.
