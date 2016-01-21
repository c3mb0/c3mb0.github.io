---
layout:     post
title:      Get Started with Pi, Fast
date:       2016-01-20
summary:    Oh look, it's another Raspberry Pi tutorial on the internet.
categories: raspberry pi
---
I recently switched back to Raspbian (they have a special distro called mini for headless systems now - yay) from OSMC when I found out that HDMI to Component conversion completely destroys image quality. During the process, I found myself digging through an endless sea of bookmarks, so once I was done, I decided to document some essentials.

Oh yeah, I'm on Windows and so are my tools. Got a problem with that? Tell that to my Cygwin, kthx.

## Tools

 * [Raspbian Image](https://www.raspberrypi.org/downloads/raspbian/)
 * [Win32 Disk Imager](http://sourceforge.net/projects/win32diskimager/files/Archive/)
 * [PuTTY](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html) (installer recommended since PuTTYgen and Pageant are neat)
 * [WinSCP](https://winscp.net/eng/download.php)

## The First SSH

So you've flashed your SD and popped it in your Pi. You now need its IP address for your first SSH. This may or may not be a trivial task depending on where your Pi and your monitor is. If you don't have a monitor nearby, or an HDMI cable, or are just plain lazy, things get a bit more bothersome. On the internet, you will find all sorts of recommendations ranging from installing 3rd party network scanning software to using _nmap_ from another computer. Screw those, man, just login to your router and check what IP it has assigned to __raspberrypi__ (default machine name that comes with Raspbian), and use that with PuTTY. Ez-pz. By the way, the default username is __pi__ and password is __raspberry__.

## Change the Default Password

{% highlight bash %}
passwd
{% endhighlight %}

## Update Everything

Your distro has probably been dormant for a while, so refresh it with this single liner:

{% highlight bash %}
sudo apt-get update && sudo apt-get upgrade && sudo apt-get dist-upgrade && sudo apt-get autoremove && sudo apt-get autoclean
{% endhighlight %}

I also install Vim after these updates, and will be using it as my text editor of choice from now on.

## Increase USB Current

If you're planning to slap a giant external drive that is powered by USB only to your Pi like I do, you will need some extra juice to spin those disks. Open up:

{% highlight bash %}
sudo vim /boot/config.txt
{% endhighlight %}

and add the following line at the end of the file:

{% highlight bash %}
max_usb_current=1
{% endhighlight %}

Without this, your external drive won't get recognized by the system.

## Assign a Static IP

You obviously don't want to wonder what IP your Pi got assigned every time you want to SSH into it, so assigning a static IP for it is the way to go. This will also come in handy later when you are configuring your router for port forwarding. Unfortunately, most popular results on Google regarding static IP assignment are out-of-date since the way of doing this was changed on May 2015 ([source](http://raspberrypi.stackexchange.com/questions/37920/how-do-i-set-up-networking-wifi-static-ip/37921#37921)). To summarize the source I've linked, simply:

{% highlight bash %}
sudo vim /etc/dhcpcd.conf
{% endhighlight %}

and add the following at the bottom of the file:

{% highlight bash %}
interface eth0
static ip_address=192.168.1.49/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 8.8.4.4
{% endhighlight %}

Obviously these are example values. The _routers_ value is the IP address of your gateway. To find out your assignable IP range, check your routers admin panel, it's usually tucked away within DHCP settings, and make sure you pick something accordingly for _ip_address_. And yes, keep the _/24_ at the end.

__Important:__ Make sure your settings are valid, or else you won't be able to get back into your Pi after rebooting and will have to start over!

## SSH Security

Next step is to tighten your SSH security before opening it up to the world. I like using public key authentication and so should you. Fire up PuTTYgen, keep all settings as they are, and press Generate. After moving your mouse over the blank area for a while, you should have your keys. Type in your _key passphrase_ and

 * Copy the bulk of text starting with _ssh-rsa_ (this is the public key) and paste/save it somewhere
 * Save the corresponding _.ppk_ (private key) by clicking on the _Save private key_ button
 * Save the OpenSSH variant of the private key by selecting _Conversions_ on top and clicking _Export OpenSSH key_

I usually keep these together in a folder for easier access later on. Now let's introduce our public key to our Pi. In your home folder, execute the following:

{% highlight bash %}
mkdir .ssh
sudo chmod 700 .ssh
vim .ssh/authorized_keys
{% endhighlight %}

Now paste in your public key. Make sure it starts with _ssh-rsa_ and is on a single line. Save and exit, then adjust the permissions as such:

{% highlight bash %}
sudo chmod 600 .ssh/authorized_keys
{% endhighlight %}

Time to tweak our SSH server settings. Open up the SSH config file:

{% highlight bash %}
sudo vim /etc/ssh/sshd_config
{% endhighlight %}

First off, change the default port. It is strongly advised that you run your SSH server on a port other than 22. A lot of automated brute force attacks specifically target this port to check if the victim's has an entry point or not. Pick 227 as your port, for example, and you will avoid most of the flood:

{% highlight bash %}
Port 227
{% endhighlight %}

It is a universal agreement that logging in as _root_ is completely unnecessary and dangerous once you have a user that has _sudo_ rights, such as the default user _pi_, so disable _root_ login:

{% highlight bash %}
PermitRootLogin no
{% endhighlight %}

As the final layer of security, disable password authentication and force private key authentication:

{% highlight bash %}
PasswordAuthentication no
{% endhighlight %}

Note that this last configuration requires you to carry your private keys around with you if you would like to login to your Pi from an arbitrary computer with SSH capabilities. The OpenSSH variant of your private key is typically used when logging in from a UNIX based system.

## Setting Up Your Dynamic DNS

So to be able to connect to your Pi, you need to know its IP address. This is easy to control when you're within your own network, but the matter is out of your control on ISP level. I'm pretty sure nowadays most ISP's actually sell static IP addresses, but why pay for one when you can workaround the issue for free?

Enter Dynamic DNS. Basically, such a service reserves a domain name for you on the internet (yes, [the entire internet](https://www.youtube.com/watch?v=Xkr5ztKiXMM)) that points to your router's address, which you can then use to access your Pi from anywhere! There are several free services out there, but I use [Duck DNS](https://www.duckdns.org/). Mostly because it's has fucking duck as their logo/mascot, and partly because I never had any problems with their service. If you would like to do the same, follow the installation instructions found [here](https://www.duckdns.org/install.jsp). You can simply follow instructions for _linux cron_. The only input I have here is to use:

{% highlight bash %}
*/5 * * * * /home/pi/duckdns/duck.sh >/dev/null 2>&1
{% endhighlight %}

instead of

{% highlight bash %}
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
{% endhighlight %}

since I remember having problems with the latter. Maybe I messed something up back then, I dunno. All I know is that the former works.

Now that our static IP, SSH server and dynamic DNS are all ready, let's reboot:

{% highlight bash %}
sudo reboot
{% endhighlight %}

## Configuring Pageant, PuTTY and WinSCP

Pageant is pretty much a lightweight helper for PuTTY and WinSCP. Remember how you've encrypted your private key with a passphrase during the key generation step? Instead of decrypting it every time you want to use it, you can keep the decrypted version of it in memory via Pageant, and Pageant will serve it to either PuTTY or WinSCP whenever needed. This is optional but useful.

Every time you launch Pageant, you need to load your key and provide your passphrase. We can make this process a bit easier by auto-loading our keys instead of selecting them manually. Find your Pageant shortcut, go to _Properties_ and append your private key's path to the end of _Target_. For example:

{% highlight html %}
"C:\Program Files (x86)\PuTTY\pageant.exe" "C:\keys\pi_private.ppk"
{% endhighlight %}

Now when you launch Pageant, it will automatically load the key and ask you for the passphrase.

Let's head over to preparing a PuTTY session for your __local__ SSH connection. In PuTTY:

 * Type in your newly configured static IP and whichever port you've chosen
 * Go to _Connection_->_Data_ and type in _pi_ as the _Auto-login username_
 * Go to _Connection_->_SSH_->_Auth_ and _Browse_ to your private key (_.ppk_)

For the __remote__ SSH session, create a separate session, keep all settings the same as above, except for the _hostname_. Use the domain you got from your dynamic DNS service instead.

For WinSCP, session settings are pretty similar:

 * Type in your static IP and dynamic DNS domain for local and remote sessions, respectively
 * For _User Name_, type in _pi_ and leave _Password_ blank
 * Go to _Advanced_->_SSH_->_Authentication_ and _Browse_ to your private key (_.ppk_)

Save your sessions.

## Port Forwarding

You will probably want to SSH from outside your home network at some point, so you should be forwarding your Pi's SSH port. I won't go into detail on how to open ports on your router, but [this](http://portforward.com/english/routers/port_forwarding/routerindex.htm) page does. It shouldn't take too long since you already have the IP and the port to use from previous steps.

Once you're done configuring it, you'll probably be like "Oh boy, this is mint, I can access my Pi from anywhere! Let me try now!". NOPE.

<span class="red">__IDIOT ALERT:__</span> You cannot use your dynamic DNS domain from within the network you've configured it for. So if your domain is pointing to your router, and you're within your router's network, the damn thing will not work. And maybe you will keep reconfiguring, rereading, retrying and destroying all the perfectly fine configuration you've made, possibly for days, wondering what the hell is going on. Like I did. Well don't. You have to be an a different network. If you would like to test the SSH connection through your dynamic DNS domain, fire up a hotspot from your phone and try it from there or something. I'd be grateful to learn about the networking logic behind this phenomenon, if anyone cares to explain.

## Auto-mount Your External Drive

Your external drive being recognized isn't enough by itself for you to be able to use it, you also need to mount it. Instead of doing it manually every time you boot your Pi, let's make it auto-mount. First, get your drive's _UUID_ and _TYPE_, then copy them somewhere:

{% highlight bash %}
sudo blkid
{% endhighlight %}

Removable media should be mounted to _/media_ and not _/mnt_ ([source](http://www.pathname.com/fhs/pub/fhs-2.3.html#MEDIAMOUNTPOINT)). Go ahead and create a folder for your external drive. Mine's name is _MONSTER_, so I do:

{% highlight bash %}
sudo mkdir /media/MONSTER
{% endhighlight %}

Since you've created the directory with _sudo_, the owner will be _root_. Let's change that:

{% highlight bash %}
sudo chown -R pi:pi /media/MONSTER
{% endhighlight %}

Time to define the auto-mount:

{% highlight bash %}
sudo vim /etc/fstab
{% endhighlight %}

Now add this this line at the end of the file. Again, this is an example, use your own information:

{% highlight bash %}
UUID=4EB410A4C310B061 /media/MONSTER ntfs auto,users,rw,uid=1000,gid=100,umask=002 0 0
{% endhighlight %}

_uid_ is the owner's ID, in this case, 1000 represents _pi_. You can see for yourself by executing _id pi_. _gid_ is the group ID, and 100 represents the _users_ group, basically opening up the drive's access to the entire OS. Finally, _umask=002_ means _rwxrwxr-x_.  Unless you have special reasons, keep them as they are, simply change your UUID, mount path and type.

__Note:__ If you are planning to access this drive over the network as a guest (see Samba configuration below) and make changes to it instead of just reading stuff off it, change the _umask_ to 000, which will set the permissions to _rwxrwxrwx_ ("sysadmins hate him!").

## Installing and Configuring Samba

I like being able to access the files on my Pi's external drive from Windows Explorer, mainly because I run all my download jobs on the Pi and streaming media from the Pi to Windows is convenient. You can also use the same thing for smart TV's and such.

Install Samba:

{% highlight bash %}
sudo apt-get install samba
{% endhighlight %}

Open up Samba's config file:

{% highlight bash %}
sudo vim /etc/samba/smb.conf
{% endhighlight %}

Windows support is disabled by default, activate it:

{% highlight bash %}
wins support = yes
{% endhighlight %}

Finally, at the very end of the file, add the following (do I need to mention this is an example?):

{% highlight bash %}
[piDownloads]
   path=/media/MONSTER/downloads
   guest ok=yes
{% endhighlight %}

The name between the brackets is how the shared folder's name will appear on the network. The rest is trivial. There are tons of other configurations online, just Google _samba_ if you would like to see more. This is the minimal amount required to open up a folder to everyone on the network.

Now reboot once more to see your auto-mount and Samba configurations working.

## That's All, Folks!

This was a bit longer than expected, but there you go. Within the next few weeks, I'll try to pump out the Part 2 for Go and Couchbase. I'm also writing a little something something regarding data analysis, curve fitting, interpolation and extrapolation. In fact, these will all come full circle in the end. Oh, I also have a couple of neat Pi things I've done that I want to write about. So many things, so little time. Stay tuned!
