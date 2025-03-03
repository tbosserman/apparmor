Some apparmor profiles. I'm putting them on github just in case anyone
else finds them useful. It's frequently not easy to find apparmor profiles
to use as a starting point, and creating them from scratch is -ahem-
"time consuming".

One may reasonably quibble about whether I was too strict or too lenient
on these profiles. Mostly, I was trying to learn how to use apparmor and
wanted to try and make my browsers not run "unconfined".

Enjoy!

### RANDOM NOTES

1. I didn't want to run everything as root so I created an apparmor.d directory
in my home directory and copied all the profiles from /etc/apparmor.d.
```
        mkdir ~/apparmor.d
	cp /etc/apparmor.d/* ~/apparmor.d/.
```
2. Make sure you have the apparmor-utils package installed.
3. Create the initial profile as a starting point. Here's an "empty" profile
you can use to start (I used aa-easyprof to generate it). Put this file in
`$HOME/apparmor.d`.
```
        #include <tunables/global>

        profile chrome /opt/google/chrome/chrome flags=(complain) {
          #include <abstractions/base>
        }
```
4. (Temporarily) remove the ratelimit on kernel printk. A better way to do
this is to use auditd, but that has it's own set of "issues".
```
        echo 0 | sudo tee /proc/sys/kernel/printk_ratelimit
        # If you have sysctl installed you could also do this:  
        sudo sysctl -w kernel.printk_ratelimit=0
```
5. Tell apparmor to load that (mostly empty) profile so it will start
complaining.
```
        sudo apparmor_parser -r ~/apparmor.d/chrome
```
6. Start collecting log lines. There are many ways to do this. This is
what I did. If you have a better way, use it! Obviously, you'll use a
different value for "-S". Look at your watch and calendar and use the
current date / time. This will let you see the messages as they spew forth,
and also put them in a file in your current directory.
```
        journalctl -S '2025-03-01 10:10:00' --system -t kernel -g chrome --follow | tee journal.out
```
7. Run chrome. You'll see a ton of log messages spewing forth (and being
written to the file journal.out).
8. Now that you have a metric buttload of "deny" lines in that journal.out
file you can use aa-genprof to start allowing stuff.
```
        aa-genprof /opt/google/chrome/chrome -d ~/apparmor.d -f ~/journal.out
```
9. Tell the kernel to reload the profile
```
        sudo apparmor_parser -r ~/apparmor.d/chrome
```
10. Lather, rinse, repeat steps 5 through 8. Don't forget to exit chrome
each time you do it. I think the kernel doesn't change the permissions
until the app restarts. I might be wrong on that, though.
10. Once you're confident that you have a good profile you want to copy
your profile to /etc/apparmor.d and set it to "enforce" mode. First,
change `flags=(complain)` to `(flags=enforce)` in your profile. Then
copy it to /etc/apparmor.d and tell the system to reload the profile:
```
        sudo cp chrome /etc/apparmor.d
        sudo apparmor_parser -r /etc/apparmor.d/chrome
```

The first time you run aa-genprof, it's gonna take you a LONG TIME to
go through all the stuff it finds. Which is why I'm giving you a sample
profile as a starting point! As you iterate, aa-genprof will steadily be
asking you for fewer and fewer responses. Eventually, you will mercifully see
that you aren't seeing anymore log messages from the kernel for things being
denied. That is, unless you decide there are some things that you don't
want to let chrome access. That's all up to you.

### THE PROFILES
**firefox** - This profile was created on Ubuntu 22.04 LTS for Firefox
version 135.0.1. I use firefox as my default browser and I've been using
this profile for awhile now in enforcing mode and it actually seems to
work the way I want it.

**chrome** - This profile was created on Ubuntu 22.04 LTS for Chrome
version 133.0.6943.141. This one is pretty strict because it will be
used in an environment where they want to be really, Really,
REALLY paranoid about what a malicious website might try to access on
a user's computer. It won't even let you peruse your own home directory.
In real life I add a line like this so users can always
peruse their own home directory and all subdirectories under it:
```
owner /home/*/** r,
```
