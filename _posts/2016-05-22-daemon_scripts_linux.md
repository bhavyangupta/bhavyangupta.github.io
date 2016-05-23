---
layout: post
title: Running a bash script as a daemon on startup in i3 WM
---

In this post I discuss how you can run a bash script in the background on startup.
The example script I use here checks the battery voltage of my laptop every 5 
minutes and sends a desktop notification if the voltage is less than a threshold 
level. I use this as a part of my i3 window manager setup.

Here's the script:

```
#!/bin/sh
notify-send "Battery Monitor" "Started"
FOLDER="/sys/class/power_supply/BAT0/"
energy_full=$(cat "$FOLDER/energy_full")
# low_threshold = 14067000 # 30 percent
low_threshold_percent=0.20
low_threshold=$(echo "$low_threshold_percent*$energy_full" | bc) 

# Check status every 5 minutes
while true; do
  energy_now=$(cat "$FOLDER/energy_now")
  energy_delta=$(($energy_full-$energy_now))
  if [ $low_threshold -gt $energy_now ]
  then
  expire_timeout=1 # ms
  notify-send -t $expire_timeout "Battery Monitor" "Battery is low. Plug in."
  fi
  sleep 300 
done
```

All that needs to be done here is to add this to my i3 config at the end:

```
exec --no-startup-id /home/bhavya/my_scripts/battery_notification.sh

```



# References 
http://blog.tunnelshade.in/2014/05/making-i3-beautiful.html
http://edunham.net/2015/02/05/making_arch_do_the_right_thing_at_startup.html
