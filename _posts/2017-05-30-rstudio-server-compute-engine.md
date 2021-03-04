---
title: "RStudio Server on Google Compute Engine"
excerpt: "Say hello to cloud computing"
tags: [Data Science, R, GCE]
comments: true
---

*Note: This tutorial has been updated occasionally to reflect changes in the setup requirements. The most recent update was October 30, 2018. Skip to [here](#summary) if you just want the post-setup summary.*

This is a how-to guide for setting up a server or virtual machine (VM) with [Google Compute Engine](https://cloud.google.com/compute/){:target="_blank"}. In addition, I'll also show you how to install [RStudio Server](https://www.rstudio.com/products/rstudio/download-server/){:target="_blank"} on your VM, so that you can perform your analysis in almost exactly the same user environment as you're used to, but now with the full power of cloud-based computation at your disposal. Trust me, it will be awesome.

## Prerequisites (read carefully)

1. Sign up for a [12-month ($300 credit) free trial](https://console.cloud.google.com/freetrial){:target="_blank"} with the Google Cloud Platform. This requires an existing Google/Gmail acount. During the course of sign-up, you should [create a project](https://cloud.google.com/resource-manager/docs/creating-managing-projects){:target="_blank"} that will be associated with billing. This is purely ceremonial at present — we're using the free trial period after all — but a billable project ID is required before gaining access to the platform.
2. Download and follow the installation instructions for the Google Cloud SDK command line utility, `gcloud` [here](https://cloud.google.com/sdk/){:target="_blank"}.

> **Tip:** Please pay proper attention to the Google/Gmail account that you use to sign up for the Cloud Platform in Step 1. (E.g. You might have two Gmail accounts, where one is your personal Gmail and the other is linked to your university email.) Needless to say, you'll want to make sure that you use the *same* account when setting up the gloud utility in Step 2. This might all sound obvious, but it has been the primary sticking point during live tutorials, where people encounter a bunch of puzzling authentication errors simply because they aren't using a consistent account.

## Introduction

So what is a [virtual machine (VM)](https://en.wikipedia.org/wiki/Virtual_machine){:target="_blank"} and why do I need one anyway? In the simplest sense, a VM is just an emulation of a computer running inside another (bigger) computer. It can potentially perform all or more of the operations that your physical laptop/desktop does, and it might have many of the same properties (from operating system to internal architecture.) The key advantage of a VM from our perspective is that very powerful machines can be "spun up" in the cloud almost effortlessly and then deployed to tackle jobs that are beyond the capabilities of your local computer. Got a big dataset that requires too much memory to analyse on your old laptop? Load it into a high-powered VM. Got some code that takes an age to run? Fire up a VM and let it chug away without consuming any local resources. Or, better yet, write the code in parallel and then spin up a VM with lots of cores (CPUs) to get the analysis done in a fraction of the time. All you need is a working internet connection and a web browser.

Another neat feature of VM's is that everyone within a team or research group can work on the *same* OS up in the cloud, regardless of their local machine types and constraints. Anyone who's ever encountered a situation where code works perfectly on their Linux/Mac/Windows machine, but fails to run on a colleague's Windows/Mac/Linux machine, will be acutely aware of what I'm talking about. In this way, VMs can be used to overcome many of the interoperability hurdles that can plague scientific/programming teamwork.

Now, with that background knowledge in mind, Google Compute Engine is part of the [Google Cloud Platform](https://cloud.google.com/){:target="_blank"} and delivers high-performance, rapidly scalable VMs. A new VM can be deployed or shut down within seconds, while existing VMs can easily be ramped up or down (cores added, RAM added, etc.) depending on a project's needs. In my experience, Google Compute Engine is at least as good as [Amazon's AWS](https://aws.amazon.com/){:target="_blank"} — say nothing of the [other really cool products](https://cloud.google.com/products/){:target="_blank"} within the Cloud Platform suite — and most individual users would be really hard-pressed to spent more than a couple of dollars a month using it. (If that.) This is especially true for the researcher who only needs to crunch a particularly large dataset or run some intensive simulations on occasion, and can easily switch the machine off when it's not being used.

> **Tip:** While I very much stand by the above paragraph, it is ultimately *your* responsibility to keep track of your billing and utilisation rates. Take a look at [Google's Could Platform Pricing Calculator](https://cloud.google.com/products/calculator/){:target="_blank"} to see how much you can expect to be charged for a particular machine and level of usage. You can even [set a budget and create usage alerts](https://support.google.com/cloud/answer/6293540?hl=en){:target="_blank"} if you want to be extra cautious.

Two final housekeeping notes, before continuing.

First, it's possible to complete nearly all of the steps in this guide via the [Compute Engine browser console](https://console.cloud.google.com/compute/instances){:target="_blank"}. However, we'll stick with the `gcloud` command line utility (which you should have [installed](https://cloud.google.com/sdk/){:target="_blank"} already), because that will make it easier to [document our steps](http://remi-daigle.github.io/shell/){:target="_blank"} and will also save us some headaches further down the road. For example, when it comes to transferring files between your local computer and a Compute Engine instance *en masse*.

Second, almost all VMs run on some variant of Linux. Since we'll only be connecting to our VM instance via the terminal, this only matters insofar as some of the commands might invoke slightly different syntax to what you'd normally use on a Mac or Windows PC. If you're brand new to Linux, then I'd recommend taking a quick look at [this website](https://linuxjourney.com/){:target="_blank"}. It provides a great step-by-step overview of some of the key concepts and commands. One thing that I'll briefly mention here is that Ubuntu — the Linux distribution that we'll be using below — uses the `apt` package-management system. (Much like macOS uses [Homebrew](https://brew.sh/){:target="_blank"}.) So when you see commands like `apt install PACKAGENAME`, that's just a convenient way to install and manage packages.

Okay, introduction out of the way. Let's get up a running.

## Setup and log-in

You'll need to choose an operating system for your VM, as well as the server zone (region). To see the available options, first open up the terminal ([Windows](http://www.digitalcitizen.life/7-ways-launch-command-prompt-windows-7-windows-8){:target="_blank"}, [Mac](https://www.techwalla.com/articles/how-to-open-terminal-on-a-macbook){:target="_blank"}, [Linux](http://www.wikihow.com/Open-a-Terminal-Window-in-Ubuntu){:target="_blank"}). Then enter (without the `~$` cursor prompt):
```
~$ sudo gcloud compute images list
~$ sudo gcloud compute zones list
```
> **Tip:** If you get an error message running the above commands, try re-running them without the "sudo" bit at the beginning. This stands for "[<b>su</b>peruser <b>do</b>](https://en.wikipedia.org/wiki/Sudo){:target="_blank"}", which invokes special user privileges as a security check, but may be redundant on your system. Clearly, if this applies to you, then you will need to do the same for any other commands invoking "sudo" for the rest of this tutorial.

I'll go with Ubuntu 18.04 and set my zone to the U.S. west coast, because that's closest to me (although it shouldn't really matter).

> **Tip:** You can set the default zone in your local client so that you don't need to specify it every time. See [here](https://cloud.google.com/compute/docs/gcloud-compute/#set_default_zone_and_region_in_your_local_client){:target="_blank"}.

You can also choose a bunch of other options by using the appropriate flags — see [here](https://cloud.google.com/sdk/gcloud/reference/compute/instances/create){:target="_blank"}. I'm going to call my VM instance "rstudio" but you can obviously call it whatever you like. I'm also going to specify the type of machine that I want. In this case, I'll go with the `n1-standard-8` option (8 CPUs with 30GB RAM), but you can choose from a [range](https://cloud.google.com/compute/pricing){:target="_blank"} of machine/memory/pricing options. (Assuming a monthly usage rate of 20 hours, this VM will only [cost about](https://cloud.google.com/products/calculator/#id=efc1f1b1-175d-4860-ad99-9006ea39651b){:target="_blank"} $7.60 a month to maintain once our free trial ends. Regardless, you needn't worry too much about these initial specs now: It's very easy to change the specs of your VM down the line and Google will even suggest cheaper alternatives if it thinks that you aren't using your resource capabilities efficiently over time.) In the terminal window, type:
```
~$ sudo gcloud compute instances create rstudio --image-family ubuntu-1804-lts --image-project ubuntu-os-cloud  --machine-type n1-standard-8 --zone us-west1-a
```

This should generate something like:
```
Created [https://www.googleapis.com/compute/v1/projects/YOUR-PROJECT/zones/us-west1-a/instances/rstudio].
NAME      ZONE        MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
rstudio  us-west1-a  n1-standard-8               10.138.0.2   104.198.7.157  RUNNING
```

Write down the External IP address, as we'll need it for running RStudio Server later.

> **Tip:** This IP address is ephemeral in the sense that it is only uniquely assigned to your VM while it is running continuously. This shouldn't create any significant problems, but if you prefer a static (i.e. non-ephemeral) IP address that is always going to be associated with a particular VM instance, then this is easily done. See [here](https://cloud.google.com/compute/docs/configure-instance-ip-addresses#assign_new_instance){:target="_blank"}.

On a similar note, RStudio Server will run on port 8787 of the External IP, which we need to enable via the Compute Engine firewall.
```
~$ sudo gcloud compute firewall-rules create allow-rstudio --allow=tcp:8787
```

> **Tip:** While I don't cover it in this tutorial, anyone looking to install and run [Jupyter Notebooks](http://jupyter.org/){:target="_blank"} on their VM should follow a similar step. Just amend the above command to Jupyter's default port of 8888.

Congratulations: Set-up for your Compute Engine VM instance is complete!

Easy, wasn't it?

The next step is to log in via [SSH](https://en.wikipedia.org/wiki/Secure_Shell){:target="_blank"}. This is a simple matter of providing your VM's name and zone (if you forget to specify the zone or haven't assigned a default, you'll be prompted):

```
~$ sudo gcloud compute ssh rstudio --zone us-west1-a
```
> **Tip:** When trying to SSH into your VM, you may get a message to effect of `Error: Please login as the user "ubuntu" rather than the user "root".` If this happens, simply amend your command as follows: `~$ sudo gcloud compute ssh ubuntu@rstudio --zone us-west1-a`

Upon logging into a Compute Engine project via SSH for the first time, you will be prompted to generate a key passphrase. Needless to say, you should **make a note of this passphrase** for future long-ins. Your passphrase will be required for all future remote log-ins to Google Cloud projects via `gcloud` and SSH from your local computer. This includes additional VMs that you create under the same project account.

Passphrase successfully created and entered, you should now be connected to your VM via terminal. That is, you should see something like the following, where "root" is your username and "rstudio" is the server hostname (i.e. your VM):

```
root@rstudio:~#
```
> **Tip:** Don't worry if you're logged in under a different username instead of "root" like I have here. Again, this is just a reflection of your OS security defaults and/or user preferences. However, it does mean that you will probably have to *add* "sudo" to the beginning of the remaining terminal commands while you are connected to your VM. In other words, reverse the earlier tweak that I suggested in this tutorial...

Next, we'll install *R*.

### Install *R* on your VM

You can find the full set of instructions and recommendations for installing *R* on Ubuntu [here](https://cran.r-project.org/bin/linux/ubuntu/README){:target="_blank"}. Or you can just follow my choices below, which should cover everything that you need.
```
root@rstudio:~# sh -c 'echo "deb https://cloud.r-project.org/bin/linux/ubuntu bionic-cran35/" >> /etc/apt/sources.list'
root@rstudio:~# apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9
root@rstudio:~# apt update
root@rstudio:~# apt install r-base r-base-dev
```
> **Tip:** Again, if you have trouble running any of the above commands — particularly if your username isn't "root" — then you should try to add "sudo" to the beginning of each line. Continue doing this for the remaining commands below as needed.

In addition to the above, a number of important *R* packages require external Linux libraries that must first be installed separately on your VM. For example, the `curl` *R* package, which in turn is a dependency for many other packages. In Ubuntu (or other Debian-based distros), run the below commands in your VM's terminal:

1) For the "[tidyverse](http://tidyverse.org/){:target="_blank"}" suite of packages:
```
root@rstudio:~# apt install libcurl4-openssl-dev libssl-dev libxml2-dev
```
2) For the main spatial libraries (sp, rgeos, sf, etc.):
```
root@rstudio:~# add-apt-repository -y ppa:ubuntugis/ubuntugis-unstable
root@rstudio:~# apt update && apt upgrade
root@rstudio:~# apt install libgeos-dev libproj-dev libgdal-dev libudunits2-dev
```

*R* is now ready to go your on VM directly from terminal:
```
root@rstudio:~# R
```
However, we'd obviously prefer to use the awesome IDE interface provided by RStudio (Server). So that's what we'll install and configure next, making sure that we can run RStudio Server on our VM via a web browser like Chrome or Firefox from our local computer. (Hit `q()` and then `n` to exit the terminal version of *R* if you opened it above.)

## Install and configure RStudio Server

### Download RStudio Server on your VM

You should check what the latest available version of Rstudio Server is [here](https://www.rstudio.com/products/rstudio/download-server/){:target="_blank"}, but as of the time of writing the following is what you need:
```
root@rstudio:~# apt install gdebi-core
root@rstudio:~# wget https://download2.rstudio.org/rstudio-server-1.1.456-amd64.deb
root@rstudio:~# gdebi rstudio-server-1.1.456-amd64.deb
```

### Add a user

Now that you're connected to your VM, you might notice that you never actually logged in as a specific user. (More discussion [here](https://groups.google.com/forum/#!msg/gce-discussion/DYfDOndtRTU/u_3kzNPqDAAJ){:target="_blank"}.) In fact, like me you may be automatically logged into your VM as root. (Fun fact: You can tell because the command prompt is a `#` instead of a `$`.) This doesn't matter for most applications, but RStudio Server specifically requires a username/password combination. So we must first create a new user and give them a password before continuing. For example, we can create a new user called "elvis" like so:
```
root@rstudio:~# adduser elvis
```
You will then be prompted to specify a user password (and confirm various bits of biographical information which you can largely ignore). An optional, but recommended step is to add your new user to the `sudo` group. We'll cover this in more depth later in the tutorial, but being part of the `sudo` group will allow Elvis to temporarily invoke superuser priviledges when needed.
```
root@rstudio:~# usermod -aG sudo elvis
```

> **Tip:** Once created, you can now log into a user's account on the VM directly via SSH, e.g. `sudo gcloud compute ssh elvis@rstudio --zone us-west1-a`

### Navigate to the RStudio Server instance in your browser

You are now ready to open up RStudio Server by navigating to the default 8787 port of your VM's External IP address. (You remember writing this down earlier, right?) If you forgot to write the IP address down, don't worry: You can find it by logging into your Google Cloud console and looking at your [VM instances](https://console.cloud.google.com/compute/instances){:target="_blank"}, or by opening up a new terminal window (*not* the one currently connected to your VM) and typing:
```
~$ sudo gcloud compute instances describe rstudio  --zone us-west1-a
```
Either way, once you have the address, open up your preferred web browser and navigate to:
```
http://EXTERNAL-IP-ADDRESS:8787
```
You will be presented with the following web page. Log in using the username/password that you created earlier.

![]({{ site.url }}/assets/images/post-images/rstudio-server-login.png)

And we're all set. Here is RStudio Server running on my laptop via Google Chrome. (It's a slightly older version of R in this image, but that just reflects when I first wrote this post.)

> **Tip:** Hit F11 to go full screen in your browser. The server version of RStudio is then almost indistinguishable from the desktop version.

![]({{ site.url }}/assets/images/post-images/rstudio-server-open.png)

## Stopping and (re)starting your VM instance
Stopping and (re)starting your VM instance is very simple, so you don't have to worry about getting billed for times when you aren't using it. In a new terminal (not the one currently synced to your VM instance):
```
~$ sudo gcloud compute instances stop rstudio
~$ sudo gcloud compute instances start rstudio
```

## Summary

Assuming that you have gone through the initial set-up, here's the **tl;dr** summary of how to deploy an existing VM with RStudio Server:

1) Start-up a VM instance.
```
~$ sudo gcloud compute instances start YOUR-VM-INSTANCE-NAME
```
2) Take note of the External IP address if you need to (see step 4 below):
```
~$ sudo gcloud compute instances describe YOUR-VM-INSTANCE-NAME
```
3) Log-in via SSH.
```
~$ sudo gcloud compute ssh YOUR-VM-INSTANCE-NAME
```
4) Open up a web browser and navigate to RStudio Server via your VM's External IP address (enter your username/password as needed):
```
http://<external-ip-address>:8787
```
5) Stop your VM:
```
~$ sudo gcloud compute instances stop YOUR-VM-INSTANCE-NAME
```
And, remember, if you really want to avoid the command line, then you can always go through the [Compute Engine browser console](https://console.cloud.google.com/home/dashboard){:target="_blank"}.

In one sense, this tutorial could end right now. You have successfully installed all the programs and components that you'll need for high-performance statistical analysis and computing. Your VM will be ready to go with RStudio Server whenever you want it. However, there are still a few more tweaks and tips that we can use to really improve our user experience and reduce complications when interacting with these VMs from our local computers. The rest of this tutorial covers my main tips and recommendations.

---

## BONUS: Getting the most out of your Compute Engine + RStudio Server setup


### Install the Intel Math Kernel Library (MKL) or OpenBLAS/LAPACK

*R* ships with its own libraries for performing the BLAS (Basic Linear Algebra Suprograms) and LAPACK (Linear Algebra PACKage) numerical routines that are the backbone of statistical and computational programming. While this default works well enough, you can get *significant* speedups by switching to more optimized libraries such as the [Intel Math Kernel Library (MKL)](https://software.intel.com/en-us/mkl){:target="_blank"} or [OpenBLAS](https://www.openblas.net/){:target="_blank"}, which support multi-threading. The former is slightly faster according to the benchmark tests that I've seen, but was historically harder to install. However, thanks to [Dirk Eddelbuettel](https://github.com/eddelbuettel/mkl4deb){:target="_blank"}, this is now very easily done. In terminal:
```
~$ git clone https://github.com/eddelbuettel/mkl4deb.git
~$ sudo bash mkl4deb/script.sh
```
Wait for the script to finish running. Once it's done, your *R* session should automatically be configured to use MKL by default. You can check yourself by opening up *R* and checking the `sessionInfo()` output, which should return something like:
```
Matrix products: default
BLAS/LAPACK: /opt/intel/compilers_and_libraries_2018.2.199/linux/mkl/lib/intel64_lin/libmkl_rt.so
```
(NOTE: Dirk's script only works for Ubuntu and other Debian-based Linux distros. If you decided to spin up a different OS for your VM than we did in this tutorial, then you are probably better off [installing OpenBLAS](https://github.com/xianyi/OpenBLAS/wiki/Precompiled-installation-packages){:target="_blank"}.)


### Transfer and sync files between your VM and your local computer

You have three main options.

#### 1. Manually transfer files directly from RStudio Server

This is arguably the simplest option and works well for copying files from your VM to your local computer. However, I can't guarantee that it will work as well going the other way; you may need to adjust some user privileges first.

![]({{ site.url }}/assets/images/post-images/rstudio-move-cropped.png)

#### 2. Manually transfer files and folders using the command line or SCP

Manually transferring files or folders across systems is done fairly easily using the command line:

```
~$ sudo gcloud compute scp rstudio:/home/elvis/Papers/MyAwesomePaper/amazingresults.csv ~/local-directory/amazingresults-copy.csv --zone us-west1-a
```
It's also possible to transfer files using your regular desktop file browser thanks to SCP. (On Linux and Mac OSX at least. Windows users first need to install a program call WinSCP.) See [here](https://cloud.google.com/compute/docs/instances/transfer-files){:target="_blank"}.

> **Tip:** The file browser-based SCP solution is much more efficient when you have assigned a static IP address to your VM instance — otherwise you have to set it up each time you restart your VM instance and are assigned a new ephemeral IP address — so I'd advise doing that [first](https://cloud.google.com/compute/docs/configure-instance-ip-addresses#assign_new_instance){:target="_blank"}.

#### 3. Sync with Git(Hub), Box, Dropbox, or Google Drive

This is my own preferred option. Ubuntu, like all virtually Linux distros, comes with Git preinstalled. You should thus be able to sync your results across systems using Git(Hub) in the [usual fashion](http://happygitwithr.com/){:target="_blank"}. I tend to use the command line for all my Git operations (committing, pulling, pushing, etc.) and this works exactly as expected once you've SSH'd into your VM. However, Rstudio Server's built-in Git UI also works well and comes with some nice added functionality (highlighted diff. sections and so forth).

While I haven't tried it myself, you should also be able to install [Box](http://xmodulo.com/how-to-mount-box-com-cloud-storage-on-linux.html){:target="_blank"}, [Dropbox](https://www.linuxbabe.com/cloud-storage/install-dropbox-ubuntu-16-04){:target="_blank"} or [Google Drive](http://www.techrepublic.com/article/how-to-mount-your-google-drive-on-linux-with-google-drive-ocamlfuse/){:target="_blank"} on your VM and sync across systems that way. If you go this route, then I'd advise installing these programs as sub-directories of the user's "home" directory. Even then you may run into problems related to user permissions. However, just follow the instructions for linking to the hypothetical "TeamProject" folder that I describe below (except that you must obviously point towards the relevant Box/Dropbox/GDrive folder location instead) and you should be fine.

> **Tip:** Remember that your VM lives on a server and doesn't have the usual graphical interface — including installation utilities — of a normal desktop. You'll thus need to follow command line installation instructions for these programs. Make sure you scroll down to the relevant sections of the links that I have provided above.

Last, but not least, Google themselves encourage data synchronisation on Compute Engine VMs using another product within their Cloud Platform, i.e. [Google Storage](https://cloud.google.com/storage/){:target="_blank"}. This is especially useful for really big data files and folders, but beyond the scope of this tutorial. (If you're interested in learning more, see [here](https://cloud.google.com/solutions/filers-on-compute-engine){:target="_blank"} and [here](https://cloud.google.com/compute/docs/disks/gcs-buckets){:target="_blank"}.)

### Share files and libraries between multiple users on the same VM

The default configuration that I have described above works perfectly well in cases where you are a single user and don't venture outside of your home directory (and its sub directories). Indeed, you can just add new folders within this user's home directory using [standard Linux commands](https://linuxjourney.com/lesson/make-directory-mkdir-command){:target="_blank"} and you will be able to access these from within RStudio Server when you log in as that user.

However, there's a slight wrinkle in cases where you want to share information between *multiple* users on the same VM. (Which may well be necessary on a big group project.) In particular, RStudio Server is only going to be able to look for files in each individual user's home directory (e.g. `/home/elvis`.) Similarly, by default on Linux, the *R* libraries that one user installs [won't necessarily](https://stackoverflow.com/a/44903158){:target="_blank"} be available to other users.

The reason has to do with user permissions; since Elvis is not an automatic "superuser", RStudio Server doesn't know that he is allowed to access other users' files and packages in our VM, and vice versa. Thankfully, there's a fairly easy workaround, involving standard Linux commands for adding [user and group](https://linuxjourney.com/lesson/users-and-groups){:target="_blank"} [privileges](https://linuxjourney.com/lesson/file-permissions){:target="_blank"}. I won't explain these in depth here, but here's an example solution that should cover most cases:

#### Share files across users

Let's say that Elvis is working on a joint project together with a colleague called Priscilla. (Although, some say they are more than colleagues...) They have decided to keep all of their shared analysis in a new directory called `TeamProject`, located within Elvis's home directory. Start by creating this new shared directory:
```
root@rstudio:~# mkdir /home/elvis/TeamProject
```
Presumably, a real-life Priscilla would already have a user profile at this point. But let's quickly create one too for our fictional version.
```
root@rstudio:~# adduser priscilla
```
Next, we create a user group. I'm going to call it "projectgrp", but as you wish. The group setup is useful because once we assign a set of permissions to a group, any members of that group will automatically receive those permissions too. With that in mind, we should add Elvis and Priscilla to "projectgrp" once it is created:
```
root@rstudio:~# groupadd projectgrp
root@rstudio:~# gpasswd -a elvis projectgrp
root@rstudio:~# gpasswd -a priscilla projectgrp
```
Now we can set the necessary ownership permissions to the shared `TeamProject` directory. First, we use the `chown` command to assign ownership of this directory to a default user (in this case, "elvis") and the other "projectgrp" members. Second, we use the `chmod 770` command to grant them all read, write and execute access to the directory. In both both cases, we'll use the `-R` flag to recursively set permissions to all children directories of `TeamProject/` too.
```
root@rstudio:~# chown -R elvis:projectgrp /home/elvis/TeamProject
root@rstudio:~# chmod -R 770 /home/elvis/TeamProject
```
The next two commands are optional, but advised if Priscilla is only going to be working on this VM through the `TeamProject` directory. First, you can change her primary group ID to "projectgrp", so that all the files she creates are automatically assigned to that group:
```
root@rstudio:~# usermod -g projectgrp priscilla
```
Second, you can add a symbolic link to the `TeamProject` directory in Priscilla's home directory, so that it is immediately visible when she logs into RStudio Server. (Making sure that you switch to her account before running this command):
```
root@rstudio:~# su - priscilla
priscilla@rstudio:~$ ln -s /home/elvis/TeamProject /home/priscilla/TeamProject
priscilla@rstudio:~$ exit
```

##### Share *R* libraries (packages) across users

Sharing *R* libraries across users is less critical than being able to share files. However, it's still annoying having to install, say, `ggplot2` when your colleague has already installed it under her user account. Luckily, the solution to this annoyance very closely mimics the solution to file sharing that we've just seen above: We're going to set a default system-wide *R* library path and give all of our users access to that library via a group. For convenience I'm just going to contine with the "projectgrp" group that we created above. However, you could also create a new group (say, "rusers"), add individual users to it, and proceed that way if you wanted to.

The first thing to do is determine where your exisiting system-wide R library is located. Open up your *R* console and type (without the ">" prompt):
```
> .libPaths()
```
This will likely return several library paths. The system-wide library path should hopefully be pretty obvious (e.g. no usernames) and will probably be one of `/usr/lib/R/library` or `/usr/local/lib/R/site-library`. In my case, it was the former, but adjust as necessary.

Once we have determined the location of our system-wide library directory, we can recursively assign read, write and execute permissions to it for all members of our group. Here, I'm actually using the parent directory (i.e. `.../R` rather than `.../R/library`), but it should work regardless. Go back to your terminal and type:
```
root@rstudio:~# chown elvis:projectgrp -R /usr/lib/R/ ## Get location by typing ".libPaths()" in your R console
root@rstudio:~# chmod -R 775 R/
```
Once that's done, tell *R* to make this shared library path the default for your user, by adding it to their `~/.Renviron` file:
```
root@rstudio:~# echo 'export PATH="R_LIBS_USER=/usr/lib/R/library"' >> ~/.Renviron
```
The *R* packages that Elvis installs should now be immediately available to Priscilla and vice versa.

> **Tip:** If you've already installed some packages in a local (i.e. this-user-only) library path before creating the system-wide setup, you can just move them across with the ['mv'](https://linuxjourney.com/lesson/move-mv-command){:target="_blank"} command. Something like the following should work, but you'll need to check the appropriate paths yourself: `root@rstudio:~# mv "/home/elvis/R/x86_64-pc-linux-gnu-library/3.5/*" /usr/lib/R/library`.


### Other tips
Remember to keep your VM system up to date (just like you would a normal computer).
```
root@rstudio:~# apt update
root@rstudio:~# apt upgrade
```
You can also update the `gcloud` utility components on your local computer (i.e. not your VM) with the following command:
```
~$ sudo gcloud components update
```


## Additional resources

As a final word, just remember to consult the official documentation if you ever get stuck. There's tonnes of useful advice and extra tips for getting the most out of your VM setup, including ways to integrate your system with other products within the Google Cloud platform like BigQuery, Storage, etc. etc.
- Google Compute Engine documentation ([link](https://cloud.google.com/compute/docs/){:target="_blank"})
- RStudio Server documentation ([link](https://support.rstudio.com/hc/en-us/articles/234653607-Getting-Started-with-RStudio-Server){:target="_blank"})
- Linux Journey guide ([link](https://linuxjourney.com/){:target="_blank"})
