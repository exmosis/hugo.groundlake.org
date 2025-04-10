---
title: "Hacking together a Grid-Aware Backup in an hour"
description: "How I spent an hour getting the current UK energy mix to control the speed of my backups"
date: 2025-04-10T11:30:17+01:00
draft: false
---

### Intro

This is a write-up of an idea that's been noodling around my head for a few weeks, coming out of a new year's resolution to get my personal backups sorted out.

If you're a self-professed techie like me, chances are you have a lot of data "lying around", including photos, videos, music, and various VMs, docker installs and git histories. You've probably looked into various cloud storage schemes over the years, and have various forgotten hard drives lying around in desks.

Most recently, I've been moving my backups to [pCloud](https://www.pcloud.com/) as I wanted to try out a service not based in the US or run by a US company. pCloud is based in Switzerland which is nice, but sadly I didn't find their app was up to much, and - being a geek - opted to use the [rclone](https://rclone.org/) tool on the command line to transfer data instead.

At the same time, I was reading articles about grid-aware computing as a way to make technology use more environmental. In particular, the Green Software Foundation's 2022 article introducing the idea of [Carbon-aware computing](https://hackernoon.com/our-code-is-harming-the-planet-we-need-carbon-aware-design-patterns), and Ismael Velasco's [follow-up critique article](https://hackernoon.com/carbon-aware-computing-next-green-breakthrough-or-new-greenwashing) that looks at the issue at scale. (Both are recommended.)

I wanted to see if it was feasible to tie my backups in with the third approach outlined by the GSF article, regarding **Demand Shaping**, or "Running our software so it does more when electricity is clean and less when it’s dirty". 

It certainly was possible, and the rest of this post dives into how I hacked something together at the end of a Friday afternoon with a few linux scripts and cup of tea.

### Overview

I wanted to break up my scripts into three different jobs, just to help set some structure for future work. **My main objectives were:**

1. Get the current "greenness" of the UK grid, as a percentage. 
2. Make sure a backup transfer is running, using rclone.
3. Set the transfer speed used by rclone dynamically, based on the grid greenness.

By doing this I can concentrate on the specifics of each part more easily, just like decoupling code. One of the aims here is to isolate out part 1 from anything to do with backups, and move towards a local config/endpoint that other local services can use too.

"Greenness" is deliberately in quotes too - the definition of sustainability can be quite subjective (depending on timescales and other context) and keeping this separate from everything else can make decisions and changes easier later on.

### 1. Get the current grid greenness

This is the work of a few minutes really, thanks to [electricitymaps.com](https://www.electricitymaps.com/) - I signed up for a free account, which lets you access the latest energy mix data (up to the last 24 hours) for a single zone, which is perfect for this proof-of-concept. Just sign in with a code that gets emailed out, set your name and choose your region, and you're good to go.

Going to the [developer dashboard](https://portal.electricitymaps.com/dashboard) should give you an authorisation token automatically. It's then as simple as using the example code explorer that should be obvious - I just changed the "Data Type" from "Carbon Intensity" to "Power Breakdown" to get the relevant stats.

The dashboard gives example code for curl, python and JavaScript, along with a handy "Try it now" button to show the results as well. I'll use the curl option for this prototype, as my main aim is to just get the idea _working_, not pretty, and because I'm already using rclone so command-line tools won't be a blocker.

![Screenshot of the Electricity Maps Dashboard, showing sample code to fetch data from the API](/img/post/2025-04-10/electricitymaps-dashboard.png)

(I'm fairly tech agnostic, especially for this kind of experiment. Everything here is pretty simple and standard, so adapting it to your own preferred tech _should_ be fairly straightforward.)

Assuming we have bash and curl installed locally, we can paste this straight into a new script. I decided to use the `fossilFreePercentage` as my "greenness" measure rather than `renewablePercentage` for now, which means I'm taking nuclear into account, but your own sustainability preferences may apply here. A server focusing on solar-power, for instance, might choose to use one of the more selective measures instead.

I'm using the [`jq` JSON processor tool](https://jqlang.org/) to grab out the metric I want - again, just because this is to hand and quick to apply. Porting all this to another language should still be simple. And for this, I'm just spitting the figure out to a text file, `fossilFreePercentage.txt`.

```
#!/bin/bash

curl "https://api.electricitymap.org/v3/power-breakdown/latest?zone=GB" \
        -s \
        -H "auth-token: <removed>" | jq '.fossilFreePercentage' > fossilFreePercentage.txt
```

Running this (after setting appropriate file permissions) and checking the file gives us a handy number:

```
$ ./getgridmix.sh && cat fossilFreePercentage.txt
64
```

Future work for this step will be to experiment with the [Carbon Aware API](https://carbon-aware-api.azurewebsites.net/swagger/index.html) as well, to see what it offers.

### 2. Running rclone to cloud storage with remote control

Now we want to kick off our backup process, using [`rclone`](https://rclone.org/), a powerful open-source file syncing tool. The strategies for backing up data generally, and the options for rclone itself, are way outside the scope of this little write-up, so excuse me for skipping over a lot of background detail here.

(While it's not relevant directly to this post either, I'm using rclone's [in-built self-encryption](https://www.maketecheasier.com/use-rclone-crypt-encrypt-files/) to encrypt files sent to some paid-for cloud storage.)

There are two main objectives to how we run rclone at this point. First, we want to make sure it's running regularly but not more than once, and second, we want to control its speed via an external script.

To do this, there's a quick check using `pgrep` to check if `/usr/bin/rclone` is already running, and exit if so. This means we can re-run the script regularly without worrying about running it twice. (There are definitely better ways to check this, and your rclone install may vary.)

Then, we set the default transfer speed to 1 kbps using the [`--bwlimit` option](https://rclone.org/docs/#bwlimit-bandwidth-spec), and turn on the [remote control API](https://rclone.org/rc/) using `--rc`.

(There are important security considerations around opening up the remote control API - do fully understand the implications if running this in any shared or production environment, as ever.)

Here's the very simple script, which I save as `rclone_rc.sh`:

```
#!/bin/bash

# Exit if running
rclone_pid=(`pgrep -f "/usr/bin/rclone"`)

[[ -z $rclone_pid ]] && \

	/usr/bin/rclone sync -P \
		--rc \
		--bwlimit "1k" \
		"/directory/to/be/synced" \
		"rclone_target:/directory/to/sync/to" \
		&
```

This will run rsync in a very verbose mode - `-P` will report progress and `&` will keep it running on the command line while the rest of the script exits. This is largely in place for developing and debugging the setup, but we can move to more silent and alternative notifications once we're happy with the overall setup.

Similarly, we can be cleverer about the directories to sync here, but let's focus on the key parts going on. 

Running the script manually starts rclone running, albeit very slowly. Importantly, we see that the remote control API is in place:

```
NOTICE: Serving remote control on http://localhost:5572/
```

### 3. Control transfer speed based on the grid

With the two blocks above in place, we need to translate from the latest grid "greenness" to setting a bandwidth speed in rclone. My approach here is based on my own network and usage context at the moment - later work will make this more advanced.

For now, I'm adopting a simple "input => output" mapping approach, where we have 1 input (the green mix factor) and 1 output (transfer speed). For some level of selective control, I want to effectively only transfer data if the grid mix is _above_ a certain level though, so we set a minimum threshold for the mix. And conversely, as a grid mix of 100% is unlikely, I want to treat any mix percentage that's "very high" the same, which is the "Maximum % threshold" shown below.

Between these two upper and lower thresholds for the grid mix, I want to scale up the transfer speed linearly, between a minimum and maximum speed which is set manually for now, as shown in the diagram here:

![Diagram showing scale of grid mix input mapped to transfer speed output, with maximum and minimum thresholds for grid mix input](/img/post/2025-04-10/Grid-aware_backups_Inputs_Output.png)

Future work will set the output range dynamically too via scripts - for instance, I want to adjust this based on time of day, the network I'm on, and whether other devices are requesting network transfers as well. The overall goal is to build more of a systemic approach that allows for greater feedback between all the components involved, but we'll get there bit by bit :)

I switched to hacking this part together in PHP, just because I couldn't quite face doing the mapping maths in bash just yet, but there's nothing too complex going on. I'll include the current file at the end of the post, but there are a couple of important parts to tie everything together.

First, we simply read the contents of the fossilFreePercentage text file we saved earlier. 

```
$fossilFreePercent = file_get_contents('fossilFreePercentage.txt');
```

There's absolutely no reason we couldn't make the API request at this point via PHP/curl of course, but as mentioned above, I want to separate out the blocks for simplicity, and the API doesn't need updating more than once an hour.

Second, we check the minimum and maximum grid thresholds and, between them, do the maths to work out the new speed. All the thresholds are currently set up in the script, so then we calculate the range, multiply by the current grid percentage, and add the minimum speed set:

```
	$input_range = $maxGreenPercent - $minGreenPercent;
	$input_pc = $fossilFreePercent;
	$input_pc -= $minGreenPercent;
	$input_weight = $input_pc / $input_range;

	$output_range = $maxSpeed - $minSpeed;
	$output_range_weighted = $output_range * $input_weight;
	$output_range_weighted += $minSpeed;
```

And the third part is simply to call the `rclone` API to update the current transfer speed:

```
$cmd = 'rclone rc core/bwlimit rate=' . $limit_k . 'k';
`$cmd`;
```

(The actual script turns off the limit if the mix is above the threshold, but this could easily be the maximum speed too.)

### 4. Call everything together

Everything is in place now for a test run. A small debug script just calls each of the other scripts in turn to update the latest green mix figure, make sure rclone is running, and set the transfer rate accordingly:

```
#!/bin/bash

source getgridmix.sh
source rclone_rc.sh
php rc_gridmix_control.php
```

This isn't quite cron-friendly but is easy to adapt so it can be run regularly and automatically, with the caveats about outputs above.

For now, I can run it manually and see what's happening:

```
$ ./check_grid_and_rclone.sh 
Fossil free %: 62
Input range: 50
Input weight: 0.24
Setting bandwidth limit to 316kbps
```

And on the running rclone instance, we see:

```
NOTICE: Bandwidth limit set to {316Ki 316Ki}
```

Job done (ish).

### Summary

So it's not perfect by a long way, but it does work as intended (until something unexpected happens, at least), and shows it can be done easily - everything here was originally done in around an hour, or less. 

As mentioned above next steps are to adjust the minimum and maximum speeds based on network, time of day, and whether other devices are running similar jobs on the same network (cross-network control is another reason for separating out jobs). Moving to other devices _may_ mean re-writing scripts in different languages, but most of my little ecosystem has easy access to bash and cron, at least.

There's plenty of scope for tweaking and tidying, but hopefully this is some inspiration for others to have a play. I'd be very interested in thoughts on how to improve this, but also **whether this sense even makes sense at all - are there issues with any underlying and fundamental assumptions about the approach?** 

But otherwise I'll be automating the setup and getting back to my original plan of comprehensive, encrypted, off-site backups from my self-hosted ecosystem. All the joys of being a techie!


### Appendix: Controlling rclone via PHP

```
<?php

$fossilFreePercent = file_get_contents('fossilFreePercentage.txt');

// speed limits in kb
$minSpeed = 100;
$maxSpeed = 1000;

// mix limits:
// - Below min, we don't transfer at all.
// - Between min and max, we scale up between minSpeed and maxSpeed
// - Above max, we disable speed limits
$minGreenPercent = 50;
$maxGreenPercent = 100;

echo "Fossil free %: " . $fossilFreePercent;

if ($fossilFreePercent < $minGreenPercent) {
	rc_stop();
} else if ($fossilFreePercent > $maxGreenPercent) {
	rc_noLimit();
} else {
	$input_range = $maxGreenPercent - $minGreenPercent;
	echo "Input range: $input_range\n";
	$input_pc = $fossilFreePercent;
	$input_pc -= $minGreenPercent;
	$input_weight = $input_pc / $input_range;
	echo "Input weight: $input_weight\n";

	$output_range = $maxSpeed - $minSpeed;
	$output_range_weighted = $output_range * $input_weight;
	$output_range_weighted += $minSpeed;

	rc_setBwLimit((int) $output_range_weighted);
}

function rc_setBwLimit(int $limit_k) {
	echo "Setting bandwidth limit to " . $limit_k . "kbps\n";
	$cmd = 'rclone rc core/bwlimit rate=' . $limit_k . 'k';
	`$cmd`;
}

function rc_stop() {
	echo "Stopping rclone.\n";
	`rclone rc core/bwlimit rate=1k`;
}

function rc_noLimit() {
	echo "Turning speed limit off.\n";
	`rclone rc core/bwlimit rate=off`;
}
```

----

_Hello. My name's Graham Lally and I'm passionate about re-thinking how we use technology, and what it means for the generations not yet born. You can find out more about me and what I'm up to at [groundlake.org](https://groundlake.org/). I'm always up for a chat too - you can [find me on Bluesky](https://bsky.app/profile/scribe6.bsky.social), [LinkedIn](https://www.linkedin.com/in/graham-lally-21385987?originalSubdomain=uk), or [good old-fashioned e-mail](mailto:graham@groundlake.org)._

