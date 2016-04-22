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
* Specific to Linux

.. note::
    * We're talking about system level debugging. Specific applications can have their own talks.
    * I'm not going to teach you about reading tracebacks from python or javascript or java
    * These things are specific to Linux. Individual applications could easily have their own presentations

Familiarity
-----------

.. rst-class:: build

* Command-line familiarity

  - Common commands (cd, ls, ps, cat)
  - Arguments (ls -l)
  - Files and directories (/home/bkero, /usr/bin)
  - Networking (ping google.com)

* Linux familiarity

  - Processes (cron, pulseaudio)
  - Libraries
  - Open file handles

.. note::
    * I expect some command-line familiarity. Readline wizardry not required.
    * I also expect Linux familiarity. Open files, networking, process concepts are a plus.


Let's Start
-----------

  .. figure:: /_static/logride.jpg
     :align: center

     flickr.com/photos/ocarchives/5333790414/ CC-BY 2.0

.. note::
    * Where do we start? Yes, logs. There are logs all over the system and they are the first place you should look.
    * Generally when something fails, it should log the failure somewhere so that the problem will be known.

Logs
----

* Kernel logs - Hardware & low-level OS problems
* System logs - Services that perform base userland tasks
* Application log files - Even higher level processes

.. note::
    * There are lots of different kinds of log files. Kernel, system, application.
    * You'll come to associate different strings with different levels of problem
    * When tracing log files you can either go from most to least specific or vice versa

Application logs
----------------

* Root/dedicated user: /var/log or /opt/$APP/log
* Normal user: $HOME/.local/share
* apache2, mysql, Steam
* Some use syslog

.. code-block:: bash

   $ tail -1 /var/log/apache2/error.log
   [Thu Apr 21 04:56:47 2016] AH01630: client denied by server
   configuration: /foobar

.. code-block:: bash

   $ tail -1 $HOME/.local/share/Steam/logs/stats_log.txt
   [2016-04-12 16:55:47] [AppID 63710] CAPIJobRequestUserStats -
   no stats data in server response, we must be up to date

.. note::
    * There is an exception. Some smaller applications will log to 'syslog' and show up in system logs instead of their own application logs.
    * If you're writing a small application, instead of implementing logging you might want to read man 1 logger.
    * Syslog is a process running on the system to manage logs. It can do clever things like ship them over a network to a central log server

Syslog
------
* Daemon
* Multi-process + Multi-user + Multi-system
* Examples: *syslog-ng*, *rsyslog*, *metalog*
* Multiple 'levels' of entries
* Rotates + Compresses ( + slices + dices)

.. note::
    * TODO: Slapchop logo?
    * The syslog daemon is surprisingly intelligent. It has different log levels (like severity), can automatically rotate/gzip logs, and can ship/aggregate from remote hosts.
    * Your distro probably comes with this. There are different ones like metalog, syslog-ng, or rsyslog. Each has different config and feature set.
    * If you're just reading the syslog file you probably don't care about any of that.

System logs
-----------

* Services that your system runs
* Examples: *dhclient*, *X*, *cron*
* Collected to syslog or systemd

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

* Hardware problems
* Nasty memory-related userspace crashes
* Not written to a file

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

Dealing with log files
----------------------

.. rst-class:: build

* Try passing --verbose or --debug options
* Run it 'by hand' using the --foreground flag
* Look at another 'level' of log
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

.. rst-class:: build

- You're the new sysadmin
- New work order: Make web site go

- .. code-block:: bash

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
    * You've just started your new job as a sysadmin for FoobarCom. You've been tasked with enabling the serving of the new web site that the guys in webdev have been cooking up. You have a very basic setup of a fresh Debian box running apache.
    * You've found a vhost example from stackoverflow.com (good site, but not without its faults), and are copypasta-ing that for production.
    * I know you can find the problem, but let's imagine that it's 4PM, you want to get to the pub, and you close the file before you realize you forgot to fill out the path

.. slide::

    .. figure:: /_static/404.jpg
       :class: fill

       http.cat

.. note::
    * DNS is already set up, don't worry about that part.
    * It still doesn't work. You load up the site in your browser and it 404s. Huh?

Example #1
----------

* .. code-block:: bash
   
   $ ls /var/log
   apache2
   cups
   dpkg.log
   mysql.log
   
* .. code-block:: bash

   $ ls /var/log/apache2
   access.log
   error.log

* .. code-block:: bash

   $ tail /var/log/apache2/error.log
   [109283] ERROR: /var/www/REPLACEME does not exist.

.. note::
    * You know that good programs keep log files.
    * You think apache is a good program.
    * So naturally you would expect apache to have log files.
    * Since it's a system-level service you know to look for logs in /var/log. 

.. slide::

    .. figure:: /_static/logend.jpg
       :class: fill

       commons.wikimedia.org/wiki/File:Spruce_Log_on_end_(10867)-Relic38.JPG CC-SA 3.0

    .. note::
        * Okay, onto other things

Bad Program Assumptions
-----------------------

.. figure:: /_static/TODO.jpg

.. note::
    * Let's talk about bad program assumptions
    * These are the most common ones

Users + Groups
--------------

.. code-block:: bash

   $ sudo su - www-data -s /bin/sh

   $ whoami
   www-data

   $ groups
   www-data ssl-cert bin

   $ ls -lh /var/www/index.html
   -rw-r----- 1 root root 500M Apr 20 00:01 /var/www/index.html

.. note::
    * Is your process running as the right user?
    * Has it EVER ran as a different user?
    * If so, there could be wrong perms on some times.
    * More on that later.

Configuration
-------------

.. rst-class:: build

* Is your program configured correctly?
* Common order of config parsing

  - Global config file
  - Local config file
  - Environment variables
  - Command-line arguments/flags

* Config syntax checker

* .. code-block:: bash

      $ ps ax | grep apache
      429 ?        Ss     4:23 /usr/sbin/apache2 -k start

* .. code-block:: bash

      $ apache2ctl -t -D DUMP_VHOSTS

.. notes::
   * One thing you should ask yourself is if you mucked with the configs. Was it working before?
   * There's a common order of config option parsing. 
   * Some programs such as Apache have a config checker. Here's an example.

Syntax Checkers
---------------

.. rst-class:: build

* .. code-block:: bash

     $ apache2ctl -t -D DUMP_VHOSTS
     VirtualHost configuration:
     0.0.0.0:80  is a NameVirtualHost
       default server bke.ro (/etc/apache2/sites-enabled/10-bke.ro.conf:1)
       port 80 namevhost bke.ro (/etc/apache2/sites-enabled/10-bke.ro.conf:1)
       port 80 namevhost www.bke.ro (/etc/apache2/sites-enabled/10-www.bke.ro.conf:6)


* .. code-block:: bash

     $ puppet parser validate /etc/puppet/manifests/site.pp
     Warning: The use of 'import' is deprecated at
       /etc/puppet/manifests/site.pp:5. See http://links.puppetlabs.com/puppet-import-deprecation
       (at /usr/lib/ruby/vendor_ruby/puppet/parser/parser_support.rb:110:in 'import')

.. note::
    * I wanted to show some good examples here

Libraries
---------

.. rst-class:: build

* .. code-block:: text
   :emphasize-lines: 7

    $ perl
    perl: error while loading shared libraries:
      libperl.so.5.18: cannot open shared object file: No such
        file or directory

* .. code-block:: bash

    $ ldd $(which perl)
        linux-vdso.so.1 =>  (0x00007ffc7929c000)
        libperl.so.5.18 => not found
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f67950c7000)
        /lib64/ld-linux-x86-64.so.2 (0x0000560bf358a000)

* .. code-block:: bash

    $ LD_LIBRARY_PATH=/morelibs perl -e 'print "yay"; '
    yay



.. note::
    * Sometimes the problems aren't obvious from the logs
    * We have to dig deeper

strace - the system call tracer
-------------------------------

* Traces system calls
* Traces signals
* Benchmarking tool

.. code-block:: text

   $ strace -p $(pidof myprogram)

.. note::
   * Explain the syntax, tell them about to read the 'man' pages for each syscall to figure out arguments

strace - useful incantations
----------------------------

* strace -f -p $PID
* strace -e open -p $PID 2>&1 | grep $FILE
* strace -c -f -p $PID ... ^C


ltrace - the library call tracer
--------------------------------

* Traces external library calls
* Accepts filter expressions
* Similar to strace
* Identical syntax

lsof
----

* Lists open file handles (including networks, listening sockets, linked libraries)

My program is stuck, halp!
--------------------------

.. rst-class:: build

* "There is seldom a problem that can't be figured using strace and lsof"

.. note::
    * This is a saying of mine. I'm trying to turn it into a piece of wisdom.

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
