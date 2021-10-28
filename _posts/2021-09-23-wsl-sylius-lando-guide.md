---
layout: post
title:  "WSL, Sylius, Lando and PHPStorm - small guide how to create a dev enviroment"
categories: jekyll update
---
Recently when I was working on Sylius the machine of choice was a MacBook air, where the parameters were exceptional, but any kind of project rebuild was a pain. I also had a laptop with Windows 10 that was at that time collecting dust but its insides were much better than this in MacBook. Also, I just got info about WSL2 commercialized as a Linux on Windows, so long story short I gave it a chance. The initial setup was just a regular Unix-like configuration of every needed package, php and xdebug (in case of php based project). And here we are "xdebug"... the problem of WSL and its networking magic.  After quite of research, I was able to handle this issue with help of [Lando][lando-site]. Let me show you how it went.

## Prerequisite steps:

Ok so let's say that you want to run the Sylius from scratch.
For the codebase lets use the [Sylius-standard][sylius-standard], but with some minor changes in config
files there should be no problem with making it also on other Symfony projects.

On the Windows Level you should have an PHPStorm installed as an IDE used for this guide.

First you need to have a WSL installed (preferably WSL2) and a linux based system. I will use Ubuntu 18.04 (for more information of how to install please refer [Microsoft Documentation][wsl-docs]).

Next you need to have a docker desktop and docker backend for WSL installed. As this is a simple step, there should be no problem installing it. You can find the instructions on official [Docker documentation][docker-docs].

I won't be getting into packages needed on Linux distro as many depends on your own needs, but packages like `curl`, `wget` or `git` are pretty usefull :wink: (and sometimes needed, but usually the installers will tell you about missing packages).

Well last wery important thing on the WSL is the titled Lando. You can use their [installation instruction][lando-install] also to install it.

> **_NOTE:_**  I would also recommend installing `zsh` and `oh-my-zsh` for much smoother usage of shell.
Also on the Windows level - there is a much better replacement for terminal (and/or powershell) called `Windows Terminal`. You can find it and install from Windows Store.

## Lets start

So now that we have everything in place, lets start with setting up:

### 1. First lets clone the project

In this case lets use the command:

`git clone https://github.com/Sylius/Sylius-Standard.git`

In the codebase you should see `.lando.yaml` and `.lando` directory. These are the files that will help us with deploying an enviroment :smile:. You can see this project in PHPStorm.

![PHPStorm]({{ site.url }}/assets/img/phpstorm.png)


> **_NOTE:_** Tired of indexing PHARS on PHPStorm? If you have any PHP installed on your WSL distro, you can run this command: 
```
php -r '$phar = new Phar("vendor/phpstan/phpstan/phpstan.phar"); $phar->extractTo("vendor/phpstan/phpstan/phpstan.tmp");' 
&& rm vendor/phpstan/phpstan/{phpstan,phpstan.phar} && mv vendor/phpstan/phpstan/{phpstan.tmp,phpstan}
```

> **_NOTE:_** PHPStorm indexing might be quite costly process. You can exclude the `vendor` and `var` directories by clicking RMB so they won't be indexed. 

### 2. Edit some bash configs

To make this developer magic work the xdebug needs to know the address of machine, lets then add the env variable that will resolve the IP of WSL network.
Edit with any editor `~/.zshrc` file (if you are not using zsh, you may need to modify i.e. `~/.bashrc` file), and add this line:

`export LANDO_HOST_IP_DEV=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2; exit;}')`

After this you need to restart the Terminal session or source file by calling `source ~/.zshrc`.
You can also test it by calling 

`echo $LANDO_HOST_IP_DEV`

and it should return some IP address :slightly_smiling_face:

> **_NOTE:_** A small clarification about [enviroment variable][lando-env].

### 3. Lets spin up Lando

Ok, so now we should be able to run the lando without a problem. Lets start it with command

`lando start`

First start might take a while because Lando needs to download all of the needed docker images.
But after few moments you should be greeted with, oh an error:

![First start]({{ site.url }}/assets/img/missing-table.png)

ok, you don't need to worry because it is normal with first start :D

First we need to change the database url in `.env` file. 
Just find a line:
`DATABASE_URL=mysql://root@127.0.0.1/sylius_%kernel.environment%`
and change the `127.0.0.1` address to `database`.
`database` is set by lando with `host: database` line in config.

Now need to install sylius which you can find in [installation instruction][sylius-install] but because we are using Lando containers - we don't need to use `php bin/console` but `lando console sylius:install`.

And after finishing installation we also need to build frontend. For this part we have a node container which we can run with `lando yarn` command. 
First start `lando yarn install` then `lando yarn build`.

After this you should be greeted with main page by going to `http://sylius_standard.lndo.site/` on which sylius is a fully usable application.

> **_NOTE:_** If you don't remember the URL to app you can check it by calling `lando info` under `urls`.

### 4. The awfull moment of configuring XDebug 

Ok, so now we have a working application, but let's say that you also want to modify it (which is straightforward by just changing code) and step-debug it with xdebug.

For a sake of example let's open some products with any taxon and put a debugger breakpoint for example here `src/Sylius/Bundle/CoreBundle/Doctrine/ORM/ProductRepository.php`: 

![Initial debug]({{ site.url }}/assets/img/initial-breakpoint.png)

Let's first configure phpstorm to stop at first line in settings:

![First line in config]({{ site.url }}/assets/img/php-config.png)

And listen for incoming connections.

> **_NOTE:_** The debug port should be the same as in Lando configuration. Default one is port `9003`

It should stop here but at the moment we will not have any data in debug window.

![Empty xdebug]({{ site.url }}/assets/img/xdebug-empty.png)

To fix it we have to change one more setting in phpstorm or to be exact - set path mapping to server files:

![Path mapping]({{ site.url }}/assets/img/path-mapping.png)

Tick the `Use path mapping` option and set the absolute path to `/app` so it will point to main project directory. I would recommend using `sylius_standard.lndo.site` as host, because it is static adress and the localhost one has its port dynamically assigned.

Now remove the debugger option `stop at first line` and refresh the wepage.

![Working Xdebug]({{ site.url }}/assets/img/phpstorm-working-debug.png)

And here we go - the app is able to be fully step debugged.


### 5. More advanced debugs based on Sylius/Sylius project

The functions described in steps before should let you debug the application served on localhost (and lando created urls) as well as debugging the scripts when you create e.g. Spec or Behat tests.

But there are some cases that are used in Sylius/Sylius project like Javascript UI tests. These steps require you to run served test application as well as headless chrome browser.

I will show here an example of it from the Sylius/Sylius project where lando configuration is also prepared.

The configuration is very simmilar to one from Sylius/Sylius-standard with some changes. 
The path mapping is pretty much the same with pointing to main directory as `\app`


[lando-site]: https://docs.lando.dev/
[lando-install]: https://docs.lando.dev/basics/installation.html#linux
[sylius-standard]: https://github.com/Sylius/Sylius-Standard
[wsl-docs]: https://docs.microsoft.com/en-us/windows/wsl/
[docker-docs]: https://docs.docker.com/desktop/windows/wsl/
[lando-env]: https://github.com/lando/lando/issues/2540
[sylius-install]: https://docs.sylius.com/en/latest/book/installation/installation.html