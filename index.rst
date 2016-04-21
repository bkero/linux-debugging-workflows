=============================
Debugging Workflows for Linux
=============================

About
=====
Ben Kero

Topics
------

* Spelunking for logs
* Finding bad program assumptions
* Tracing program and I/O execution
* (Specific applications can their own talk)

.. note::
    * We're talking about system level debugging. Specific applications can have their own talks.

Expectations
------------

* Command-line familiarity
* Linux familiarity

.. note::
    * I expect some command-line familiarity. Readline wizardry not required.
    * I also expect Linux familiarity. Open files, networking, process concepts are a plus.


Let's Start
-----------

* TODO: include logride.jpg
* https://www.flickr.com/photos/ocarchives/5333790414/ CC-BY 2.0

.. note::
    * Where do we start? Yes, logs. There are logs all over the system and they are the first place you should look.
    * Generally when something fails, it should log the failure somewhere so that the problem will be known.

Logs
----

* Kernel logs - Hardware and low-level OS problems (kernel)
* System logs - Services that perform higher-level tasks
* Application log files - Even higher level processes

.. note::
    * There are lots of different kinds of log files. Kernel, system, application.
    * You'll come to associate different strings with different levels of problem
    * When tracing log files you can either go from most to least specific or vice versa

Application logs
----------------

* Typically log to /var/log or /opt/$APP/log if they're run system-wide
* If run as the user, might be logged in $HOME/.local/share
* Examples: apache2, mysql, Steam
* Sometimes they can log to the syslog

.. code-block:: bash

   $ tail -1 /var/log/apache2/error.log
   [Thu Apr 21 04:56:47 2016] AH01630: client denied by server
   configuration: /foobar

.. code-block:: bash

   $ grep failed /var/log/elasticsearch/elasticsearch.log.*
   [2015-06-09 01:00:26,394][WARN ][discovery.zen.ping.multicast]
   [elasticsearch1] failed to read requesting data from /10.0.3.1:54328

.. note::
    * There is an exception. Some smaller applications will log to 'syslog' and show up in system logs instead of their own application logs.
    * If you're writing a small application, instead of implementing logging you might want to read man 1 logger.
    * Syslog is a process running on the system to manage logs. It can do clever things like ship them over a network to a central log server

System logs
-----------

* Examples: *dhclient*, *X*, *cron*
* Often globbed together

.. code-block:: bash
    :caption: not systemd

    $ tail -3 /var/log/syslog
    Apr 21 05:05:17 bkero-general dhclient: DHCPACK of 10.0.3.227 from 10.0.3.1
    Apr 21 05:05:17 bkero-general dhclient: bound to 10.0.3.227 -- renewal in 1265 seconds.
    Apr 21 05:17:01 bkero-general CRON[33412]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)

.. code-block:: bash
    :caption: systemd

    $ journalctl -xn 3
    Apr 20 22:12:21 Whitefall libvirtd[1011]: ethtool ioctl error: No such device
    Apr 20 22:12:23 Whitefall org.gnome.OnlineAccounts[1143]: (goa-daemon:1331): GoaBackend-WARNING : secret_password_lookup_sync() returned NULL
    Apr 20 22:12:24 Whitefall NetworkManager[519]: <info>  (docker0): link disconnected (calling deferred action)


.. note::
    * These are a little tricker to find. If you're running a newer systemd-based distro 

Kernel logs
-----------

* These are things like hardware problems or really bad memory-related userland crashes
* If you suspect a hardware problem, good to start here first

.. code-block:: bash

    $ dmesg | tail -1
    [05196] md: data-check of RAID array md0
    [05196] md: minimum _guaranteed_  speed: 1000 KB/sec/disk.
    [05196] md: using maximum available idle IO bandwidth (but not more than 200000 KB/sec) for data-check.
    [05196] md: using 128k window, over a total of 1953381888k.
    [25075] md: md0: data-check done.
    [34705] test[27435]: segfault at fffffffffffffffb ip 00007f9e3b...
    error 5 in libc-2.23.so[7f9e3bf6b000+198000]

.. note::
    * Every system is split into kernel-land and user-land.
    * Note the segfault. This is actually an application problem, but it's so nasty that it shows up here since the kernel is blocking it.
    * Here note the format: the first number (in braces) is seconds since the system booted. The second word is the kernel system or process. The third is the message.
    * Look at the man page for 'dmesg'. Lots of cool options including a wait.

Some Tips for dealing with log files
------------------------------------

* Try passing --verbose or --debug options
* Run it 'by hand' with something like --foreground
* Try moving from application -> system level or system -> kernel
* tail -f (dmesg -w) is your friend
* You don't live in a vacuum

.. note::
    * Many applications support multiple levels of verbosity including levels like INFO, WARN, ERROR, or FATAL
    * Many applications that run in the background (like puppet) can be run in 'test' mode in the foreground. Often times this automatically turns on debug output.
    * Try running apache with -X
    * Some log messages might be vague. In that case try moving to a different log level, or start stepping through.
    * tail -f tails logs. Pressing 'enter' can help segment things (LIVE DEMO)
    * You don't live in a vacuum. Search for your error messages online. Try to pay attention to the distro and age of the resulting posts.

Example #1
----------

* You're the new sysadmin
* New work order: Make web site go
* Here is the web site contents

Example #1
----------

.. code-block:: bash

   $ cat /etc/apache2/sites-enabled/001-foobar.com.conf
   <VirtualHost 0.0.0.0:80>
       ...
       ServerName foobar.com
       ServerAlias www.foobar.com
       ...
       DocumentRoot /var/www/REPLACEME
       ...
   </VirtualHost>
   $ sudo service apache2 restart
   ... [ OK ]

.. note::
    * Let's start with this example. You've just started your new job as a sysadmin for FoobarCom. You've been tasked with enabling the serving of the new web site that the guys in webdev have been cooking up. You have a very basic setup of a fresh Debian box running apache.
    * You've found a vhost example from stackoverflow.com (good site, but not without its faults), and are copypasta-ing that for production.
    * I know you can find the problem, but let's imagine that it's 4PM, you want to get to the pub, and you close the file before you realize you forgot to fill out the path

Example #1
----------

* TODO: Pic of 404

.. note::
    * DNS is already set up, don't worry about that part.
    * It still doesn't work. You load up the site in your browser and it 404s. Huh?

Example #1
----------

* Good programs keep log files
* Apache is a good program (*)
* You expect Apache to have log files

.. code-block:: bash
   
   $ ls /var/log
   ...apache2...
   $ ls /var/log/apache2
   ...error.log...
   $ tail /var/log/apache2/error.log

* YOU MADE A TYPO YOU JERK

.. note::
    * You know that good programs keep log files. You think apache is a good program. So naturally you would expect apache to have log files. Since it's a system-level service you know to look for logs in /var/log. 

Bad Program Assumptions
-----------------------
* Are you running as the right user? (whoami)
* Are you passing the program weird options (ps)
* Are you running the right library version? (ld)

.. note::
    * Sometimes the problems aren't obvious from the logs
    * We have to dig deeper

I don't know what's wrong, halp
-------------------------------

* Let's trace it!

strace - the system call tracer
-------------------------------

ltrace - the library call tracer
--------------------------------

lsof
----

* Lists open file handles (including networks, listening sockets, linked libraries)

gdb
---
* The big guns, step-through debugger

systemtap
---------
* The debugging suite
* Has its own programming language
* Lots of cargo-cult scripts, (yay dtrace!)

lttng
-----
* Linux tracer toolkit Nextgen
* Created to trace things that were difficult to trace otherwise
* Good for solving perf problems
* Kernel-based, out-of-tree. Installing packages requires kernel modules


The problem must be somewhere else
----------------------------------
* Network sniffing!
* tcpdump for command-line
* Wireshark for record/analyze streams/replay
