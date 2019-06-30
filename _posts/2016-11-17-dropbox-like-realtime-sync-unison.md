---
layout: post
category: posts
title: Dropbox-like real-time file synchronization with Unison
comments: true
---

Dropbox is a cloud service that allows for real-time synchronization of files between multiple
devices. While it offers a lot of advantages, some people prefer to have control over their data,
specifically when privacy concerns arise. Self-hosted alternatives offer advantages such as

- Full control over your data. For example, it can be stored on servers outside of the USA.
- Upload/download speed might improve significantly.
- Depending on your setup, there won't be charges for additional storage.

Here, I present a method which uses state-of-the-art Linux software only, to accomplish real-time
file synchronization using a central server. With "real-time" I mean the immediate synchronization
after files have changed on either side. For this task, we are going to use
[Unison](https://www.cis.upenn.edu/~bcpierce/unison/), [SSH](https://www.openssh.com/) and
[systemd](https://www.freedesktop.org/wiki/Software/systemd/). In the following tutorial we're going
to distinguish between commands on the client and server by the respective `user@...` bash prompts.

#### Key-based, password-less server access

In order to synchronize files in the background, it is necessary to setup automated access to the
central server. Generate a SSH key using

{% highlight bash %}
user@client$ ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_share
{% endhighlight %}

Keep an empty passphrase when asked. For security reasons, it is highly advised to create
a dedicated user on your server. For the remainder of this tutorial, let's call it `share`:

{% highlight bash %}
user@server$ sudo adduser share
user@server$ sudo passwd share
{% endhighlight %}

Now copy the generated public key to the server:

{% highlight bash %}
user@client$ ssh-copy-id -i ~/.ssh/id_rsa_share.pub share@server
{% endhighlight %}

#### Installing Unison

We have to install unison on __both the client and the server__. As we make use of Unison's latest
`fsmonitor` feature which detects changed files, we have to download and compile Unison ourselves.
But first let's install ocaml and inotify for Python as Unison dependencies:

{% highlight bash %}
user@client/server$ sudo dnf install ocaml python-inotify # Fedora
user@client/server$ sudo apt-get install ocaml python-pyinotify # Debian / Ubuntu
{% endhighlight %}

At the time of writing, 2.48 was the latest stable version of Unison. Let's get it from SVN and
compile/install.

{% highlight bash %}
user@client/server$ svn co https://webdav.seas.upenn.edu/svn/unison/branches/2.48 unison
user@client/server$ cd unison
user@client/server$ make NATIVE=false UISTYLE=text
user@client/server$ sudo cp src/{unison,fsmonitor.py} /usr/local/bin
{% endhighlight %}

#### Creating a Unison profile

Create a file `~/.unison/share.prf` with the following content:

{% highlight plain %}
root = ssh://share@server//home/share/unison
root = /home/<USER>/share

sshargs = -oIdentityFile=/home/<USER>/.ssh/id_rsa_share

auto = true
batch = true
perms = 0
ui = text
confirmmerge = false
confirmbigdel = false
prefer = newer
silent = false
times = true
repeat = watch
logfile = /home/<USER>/.unison.log

ignore = Name {.mypasswords.kdbx.lock}
{% endhighlight %}

This profile tells unison to share the directories specified with `root` using the generated SSH
key. It also tells Unison to not ask any questions and operate in a fully automated way. The `repeat
= watch` instructs to repeat synchronization as soon as something on either the client or the server
has changed. The `ignore` directive tells Unison to ignore certain files such as the lock file for
my password manager which is [KeepassX](https://www.keepassx.org/). These options are optimal for
myself, however you might want to adapt it to your needs. Please refer to the [Unison
manual](https://www.cis.upenn.edu/~bcpierce/unison/download/releases/stable/unison-manual.html) for
more information.

Don't forget to create the directories on the client and server:

{% highlight bash %}
user@client$ mkdir /home/<USER>/share
{% endhighlight %}

{% highlight bash %}
user@server$ mkdir /home/share/unison
{% endhighlight %}

Be sure to replace `<USER>` with your client user name in the above commands.

Now verify that the synchronizations is working by running

{% highlight bash %}
user@client$ unison share
{% endhighlight %}

and creating/modifying/deleting files in the client and server folders. Stop Unison again, as we
will create a service for running it.

#### Creating a systemd user service

Now that the synchronization is working, we create a systemd service on the client which takes care
of automatically (re-starting) unison. Create a file

`~/.config/systemd/user/unison@.service`:

{% highlight plain %}
[Unit]
Description=Unison

[Service]
Environment="PATH=/usr/local/bin:/usr/bin"
ExecStart=/usr/local/bin/unison %i
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
{% endhighlight %}

This tells systemd to start the command specified with `ExecStart` using the previously defined
`Environment` variables. In case of failure (i.e. because of a missing network connection), it
automatically restarts after 10 seconds. Here, `%i` acts as a placeholder for the Unison profile. Be
aware that the `Environment` variable contains the `PATH` to the `unison` and `fsmonitor.py`
executables, as well as for all other needed programs such as SSH.

Verify that it's working:

{% highlight bash %}
user@client$ systemctl --user start unison@share
user@client$ systemctl --user status unison@share
{% endhighlight %}

The second command should show something like `active (running)` in its output. Let's enable the
service at system startup:

{% highlight bash %}
user@client$ systemctl --user enable unison@share
{% endhighlight %}

#### Summary

This setup automatically synchronizes a directory between clients and a central server. There are
a few drawbacks (and possible solutions) to consider:

- A server is needed. If you don't have access to a server with Unison, this setup won't be possible.
- No mobile access. Easy read access might be possible though using a public directory served by
  a web-server (with authentication of course).
- No notification in case of failures. It is possible that Unison fails because of a non-resolvable
  conflict. Here, one could regularly parse the log file, detect such events and notify the user.

So far, this setup works pretty reliably for me :)
