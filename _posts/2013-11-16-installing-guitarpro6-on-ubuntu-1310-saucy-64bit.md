---
layout: post
title: "Installing Guitar Pro 6 on Ubuntu 13.10 Saucy 64-bit"
tags: [Ubuntu]
---
A few days ago I downloaded the last release of [Guitar Pro 6](http://www.guitar-pro.com/) for Linux. What first annoyed me was that there is no package for 64 architectures. Yes, you read it correctly, it's 2013 and Arobas Music still thinks that it is acceptable to provide [32-bit software only](http://www.guitar-pro.com/en/index.php?pg=support-frequent-asked-questions&faq=tech#Q19). I thought no company could to be hit by the [Unix 2038 problem](http://en.wikipedia.org/wiki/Year_2038_problem), but now it looks like we have a candidate!

<!--break-->

Tried to install the package anyway, confident that Ubuntu would do a good job installing the required 32-bit library dependencies. Unfortunately Guitar Pro 6 lists gksu among the dependencies, and that cannot be installed along with the 64-bit version already in the system.

![Cannot install gksu:i386](/img/posts/gp6-cannot-install-gkgu.png)

Googling to see how others have solved the issue, I found a [script on the Ubuntu forums](http://ubuntuforums.org/showthread.php?t=636724) that updates the control file in the package to remove the nasty dependency.

Upon installation of the new package, [Lintian](http://lintian.debian.org/) warned me that the package failed its quality checks.

![The package is of bad quality](/img/posts/gp6-package-bad-quality.png)

I could have fixed the package name with the previous script, but further checks made me find something even worse: two OSX control files (._.DS_Store and .DS_Store) that would have been installed in the root directory!

Decided then to drop the script and do it manually:

	dpkg-deb -x gp6-full-linux-r11553.deb gp6-tmp
	dpkg-deb --control gp6-full-linux-r11553.deb gp6-tmp/DEBIAN
	vi gp6-tmp/control # rename GuitarPro6 to guitarpro6, remove gksu dependency
	rm gp6-tmp/.DS_Store gp6-tmp/._.DS_Store
	dpkg -b gp6-tmp gp6-full-linux-r11553-tastethedifference.deb

Finally I had a package that was of decent quality, so I was able to install, run and register the software. Then of course the next thing I did was to run the updater. The update went all right, and I started the software again only to find out that something else went wrong:

	./GuitarPro: /opt/GuitarPro6/./libz.so.1: version `ZLIB_1.2.3.3' not found (required by /usr/lib/i386-linux-gnu/libxml2.so.2)

This was easy to fix by renaming libz.so.1 to libz.so.1.orig, but then on the next run it stopped with a white translucent window and crashed after a few seconds:

	QGtkStyle was unable to detect the current GTK+ theme.
	Segmentation fault (core dumped)

Thanks to a [blog post in the Fedora forums](http://forums.fedora-fr.org/viewtopic.php?pid=525293), I was able to identify the package to install and fix this final issue by executing

	sudo apt-get install gtk2-engines:i386

At this point the only problem is a wierd message window at startup: "gp build with Qt : 4.6.3 and run with Qt : 4.6.2." I have already done far too much to be able to run this software, so I'll have to live with it.

Don't get me wrong: I am extremely grateful to Arobas Music for providing a Linux build of their software. What annoys me is that with very little effort they could improve a lot the user experience.

