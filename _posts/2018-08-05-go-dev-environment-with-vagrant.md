---
layout: single
title: "Setting Up a Development Environment for Go"
---

Setting up a Go Development Environment
No matter what you do or where you work, new technologies are constantly coming along that change (for better or worse) what you do. Languages, frameworks, SRE stuff, CI pipelines, it seems almost endless.

Another equal and opposite fact of life is resistance to these new things. Now, this isn’t a criticism: if it’s not broke, don’t fix it is a perfectly reasonable point of view, but sometimes you have to take the chance on something new.

One of the ways I’ve found that helps address concerns when introducing a new language or framework is to set up a shareable development environment with Vagrant. Vagrant is just a system to provision virtual machines to share development environments. Whatever we set up here may not end up being another dev’s main setup, but it’s a good way to allow someone to get up and running with a full toolset quickly. So, that’s what we’re going to be doing today: setting up a Go development environment with Vagrant. 

### Prereqs
 - Vagrant installed on your machine

### Getting Started
To begin with, we're going to need to create a Vagrantfile. The easiest way to do this is with the vagrant init command:
```bash
vagrant init -m ubuntu/bionic64
```

The -m option generates a minimal Vagrantfile. If we don't use this, the default Vagrantfile that gets generated will have a ton of commented out lines. We should end up with something that looks like this:

 ```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
end
```

Ok, so we're off to a good start. The next thing we need to do is set up our provisioner. Vagrant supports a bunch of different provisioners (Chef, Puppet, etc.), but we don't need anything that fancy so, we're just going to use an inline shell provisioner. To do this, we just add the provision block inside of main Vagrant block. In other words, make your Vagrantfile look like this:

<pre>
<code>
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  <b>config.vm.provision 'shell', privileged: false, inline: <<-SHELL</b>

  <b>SHELL</b>
end
</code>
</pre>

The only mildly interesting thing here is the `privileged: false` flag. What we're here is that Vagrant shouldn't run this script as root. This is to make it easier to set up the 'vagrant' user's environment, instead of trying to set everything from root. If we need more privileges, we can just sudo individual commands.

This is all well and good, but what exactly do we need for our Go environment? Well, we'll need the Go language and toolset, plus a debugger (delve). We'll also want to set up Vim with some plugins to make working with Go code easier and then configure vimrc to make everything look nice a pretty.

Note: I'm installing version 1.8.3 of Go here (because, that's what a lot of our current production Go code is written against). If you want a different version, just substitute in the correct tar.gz url.

First things first, lets update apt and install git, curl, and Go:

<pre>
<code>
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.provision 'shell', privileged: false, inline: <<-SHELL

    <b>sudo apt-get update</b>
    <b>sudo apt-get install -y git curl</b>
    
    <b>sudo wget https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz -nv</b>
    <b>sudo tar -C /usr/local -xf go1.8.3.linux-amd64.tar.gz</b>
  SHELL
end
</code>
</pre>

Next up, let's set up our Go environment variables. Notice how we're going about this in a little bit of a round about way: instead of just calling export yada yada from our provisioning script (which would set the env vars for the provisioning stage, not the user that will login and use our environment), what we're doing is pushing the export commands into ~/.profile. This is essentially a script that runs when the users logs in.

<pre>
<code>
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.provision 'shell', privileged: false, inline: <<-SHELL

    sudo apt-get update
    sudo apt-get install -y git curl
    
    sudo wget https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz -nv
    sudo tar -C /usr/local -xf go1.8.3.linux-amd64.tar.gz

    <b>echo 'export GOPATH=/home/vagrant/go' >> ~/.profile</b>
    <b>echo 'export GOROOT=/usr/local/go' >> ~/.profile</b>
    <b>echo 'export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin' >> ~/.profile</b>

    <b>echo 'go get github.com/derekparker/delve/cmd/dlv' >> ~/.profile</b>
  SHELL
end
</code>
</pre>

Looking good. Now, let's get our Vim plugins set up. We're going to be using Pathogen to manage these plugins because it just makes things so simple. The plugins we're going to be using are NERDTree (to make it easier to navigate between files), Neocomplete (an autocomplete thing, I don't use it much, but it's pretty cool), and vim-go (a masterpiece of a plugin. Seriously, I stopped using a full on IDE when I discovered this plugin). To install these, all you need to do is clone the repos, and then symlink them to /home/vagrant/.vim

<pre>
<code>
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.provision 'shell', privileged: false, inline: <<-SHELL

    sudo apt-get update
    sudo apt-get install -y git curl
    
    sudo wget https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz -nv
    sudo tar -C /usr/local -xf go1.8.3.linux-amd64.tar.gz

    echo 'export GOPATH=/home/vagrant/go' >> ~/.profile
    echo 'export GOROOT=/usr/local/go' >> ~/.profile
    echo 'export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin' >> ~/.profile

    echo 'go get github.com/derekparker/delve/cmd/dlv' >> ~/.profile

    <b>mkdir -p ~/.vim/autoload && curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim</b>

    <b>git clone https://github.com/fatih/vim-go.git /home/vagrant/bundle/vim-go</b>
    <b>git clone https://github.com/scrooloose/nerdtree.git /home/vagrant/bundle/nerdtree</b>
    <b>git clone https://github.com/Shougo/neocomplete.vim.git /home/vagrant/bundle/neocomplete.vim</b>

    <b>ln -s /home/vagrant/bundle /home/vagrant/.vim</b>
  SHELL
end
</code>
</pre>
We're getting close. All we have left to do is set up our vimrc file to make our text look nice and our plugins behave.

<pre>
<code>
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.provision 'shell', privileged: false, inline: <<-SHELL

    sudo apt-get update
    sudo apt-get install -y git curl
    
    sudo wget https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz -nv
    sudo tar -C /usr/local -xf go1.8.3.linux-amd64.tar.gz

    echo 'export GOPATH=/home/vagrant/go' >> ~/.profile
    echo 'export GOROOT=/usr/local/go' >> ~/.profile
    echo 'export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin' >> ~/.profile

    echo 'go get github.com/derekparker/delve/cmd/dlv' >> ~/.profile

    mkdir -p ~/.vim/autoload && curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim

    git clone https://github.com/fatih/vim-go.git /home/vagrant/bundle/vim-go
    git clone https://github.com/scrooloose/nerdtree.git /home/vagrant/bundle/nerdtree
    git clone https://github.com/Shougo/neocomplete.vim.git /home/vagrant/bundle/neocomplete.vim

    ln -s /home/vagrant/bundle /home/vagrant/.vim

    <b>echo -n > .vimrc</b>
    <b>echo 'execute pathogen#infect()' >> .vimrc</b>
    <b>echo 'syntax on' >> .vimrc</b>
    <b>echo 'filetype plugin indent on' >> .vimrc</b>
    <b>echo 'let g:neocomplete#enable_at_startup = 1' >> .vimrc</b>
    <b>echo 'set number' >> .vimrc</b>
    <b>echo 'let g:go_highlight_functions = 1' >> .vimrc</b>
    <b>echo 'let g:go_highlight_methods = 1' >> .vimrc</b>
    <b>echo 'let g:go_highlight_structs = 1' >> .vimrc</b>
    <b>echo 'let g:go_highlight_operators = 1' >> .vimrc</b>
    <b>echo 'let g:go_highlight_build_constraints = 1' >> .vimrc</b>
    <b>echo 'let g:go_disable_autoinstall = 0' >> .vimrc</b>

  SHELL
end
</code>
</pre>

