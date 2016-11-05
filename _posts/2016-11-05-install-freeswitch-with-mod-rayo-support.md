---
layout: post
title:  "Install freeswitch with mod_rayo support on Ubuntu"
date:   2016-11-05 11:11:00 +0000
categories: freeswitch rayo install linux
---

Here's what I did on a fresh Ubuntu 16.04 desktop installation. I referred to the [Freeswitch wiki](https://freeswitch.org/confluence/display/FREESWITCH/FreeSWITCH+Explained)

1. In my `~/dev` directory, `git clone...`
2. `cd freeswitch`
2. `./bootstrap.sh`
3. Got "You need libtool vn xxx..."
4. `sudo apt-get install libtool-bin`
5. Repeated step 2: this time it worked
6. In the generated `modules.conf`, uncommented the `event_handlers/mod_rayo` and `formats/mod_ssml` lines
7. `./configure`
8. Got error: `no usable libjpeg; please install libjpeg devel package or equivalent`
9. `sudo apt-get install libjpeg-dev`
10. Repeated step 7 `./configure`
11. Got a long message about libcurl missing
12. `sudo apt-get install libcurl3-dev`
13. Repeated step 7 `./configure`
14. Got message about missing `libldns-dev` or disable `mod_enum` in `modules.conf`
15. `sudo apt-get install libldns-dev`
16. Repeated step 7 `./configure`
17. Got message about missing `libedit-dev`
18. `sudo apt-get install libedit-dev`
19. Repeated step 7 `./configure`
20. Some time later (about 5 mins): success
21. `make`
22. Got message: "Nether yasm nor yasm have been found. See the README..."
23. The README appears to be empty. I Googled a bit and did
24. `sudo apt-get install nasm` and `sudo apt-get install yasm`
25. Repeated step 21 `make`
26. Got much further and started compiling codecs (or something)...
27. `make` failing with error messages to install a few more packages. Something to note is that if  the build fails at this stage, and you need to do `sudo apt-get install <xxx>`, then you need to re run `./configure` and build again. I ended up commenting out `mod_lua` from `modules.conf`. At some point I may go back and re-add it. Eventually, `make` succeeded.
28. `sudo make install`
29. `cd /usr/local/freeswith/bin`
29. Started Freeswitch in the foreground with `./freeswitch -c`

See my next post for details of getting Adhearsion to connect via `mod_rayo`.
