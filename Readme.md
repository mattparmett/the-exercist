# The Exercist

Since getting an Apple Watch, I've had a lot of fun exploring the different ways that data, logging, and tracking can be used to improve day to day life.  The watch is definitely a motivating factor in terms of exercising (closing the rings can be pretty addicting), and I've gotten into looking at my "net calories" (food calories - baseline calorie burn - exercise calories) at various points of the day.

After working out the ways that the stock device / apps collect, display, and process data, I began looking at novel ways to use all the data my watch collects.  One thing I thought would be cool is to block internet access to certain time-wasting sites (facebook, twitter, reddit, etc.) until I've exercised at least once that day.  That way, I'm either motivated to exercise to be able to get online, or if I don't exercise, the time I can spend on distracting sites is limited.

## Process Overview

Setting this all up was a bit more difficult than I had anticipated, but ultimately I think I found a pretty good process.  The below outline summarizes the process, delineated by device:

- Apple Watch / iPhone

  - Log exercise on Apple Watch through Strava and Apple's Workouts app
  - Exercise logged on watch syncs to Apple Health on iPhone
  - MyFitnesspal on iPhone syncs with the Health app and uploads summary workout data to MyFitnesspal profile

- Raspberry Pi:

  - Every ten minutes, runs a script that:
    - Checks if I'm at home - determined if my iPhone is on my home wifi network (see "Router" below)
      - If I'm not at home, the script exits - no need to worry about restricting/allowing access if I'm not on my wifi
    - If I'm at home, checks MyFitnesspal to determine if I've logged any exercise aside from the iOS steps adjustment
      - If I have logged exercise, tell the router to unblock internet access
      - If there is no exercise logged, tell the router to block internet access

- Router:

  - The router needs to serve a couple of functions in this system:
    - When asked, tell Raspberry Pi if my iPhone is on the network
    - Block/unblock certain domains as requested by the Raspberry Pi exercise script


  - Install LEDE open router firmware - this gives us access to the router over SSH so the Raspberry Pi can send it commands
  - LEDE has a package called simple-adblock that allows us to block certain domains

## Setup - Apple Watch / iPhone

The setup of my watch / phone to track exercise was fairly straightforward.  Because there is a good python package to retrieve exercise data from MyFitnesspal, the ultimate goal is to automatically track exercise and get that data into MyFitnesspal so we can use it elsewhere in the process.

This part of the workflow depends on how you track workouts and fitness, but my setup is outlined below:

- Install MyFitnesspal, and set up to sync exercise data with Apple Health (this way, any workouts that are logged to Health will be mirrored into your MFP profile)
- I capture runs through Strava, which is linked up to Apple Health
- For other exercise, I generally use Apple's Workout app on my watch, which syncs to Health automatically

## Setup - Server / Router

I have a RPi that is already running as a server 24/7 for things like homebridge and a custom daily email newsletter script, so I figured I would use it to automatically check my MFP profile and send instructions to my router.

### MyFitnesspal Monitoring

There is a great MFP client written for python, `python-myfitnesspal`.  This library allows us to pull down our exercise data for the day with a pretty simple script.  After installing the library (`pip install python-myfitnesspal`), I set the client up to use my MFP account (`myfitnesspal store-password $USERNAME`).  That allows us to avoid storing/updating the password in the code.

I then wrote up a simple function to determine if I've exercised for a given date.  The function downloads exercise data from MFP and looks at both the Cardio and Strength Training sections to determine if there are any exercises logged (aside from the iOS calorie adjustment from steps logged by my watch/phone outside of workouts):

```python
def has_exercised(date):
	has_exercised = False
	mfp_exercise = mfp.get_exercise(date.year, date.month, date.day)

	cardio = mfp_exercise[0]
	for cardio_exercise in cardio:
		if cardio_exercise.name != 'MFP iOS calorie adjustment':
			has_exercised = True

	strength = mfp_exercise[1]
	if len(strength) > 0:
		has_exercised = True

	return has_exercised
```

I know that I want to block sites if I haven't exercised and unblock sites if I have exercised, which is written as the following procedure:

```python
today = datetime.date.today()
if not(has_exercised(today)):
	print "No exercise detected, blocking web access..."
	block_sites() 
else:
	print "Exercise detected, unblocking web access..."
	unblock_sites()
```

We haven't yet written the router portion of the workflow, which will be called in the `block_sites` and `unblock_sites` functions, so we'll leave those for later.

### Router Interaction

Once the RPi knows if I've exercised, it will need to give the router the appropriate direction - block access if I haven't exercised, or unblock access if I have exercised.  And, in order to prevent unnecessary action on the router side of things, I'd like to prevent the process from even running if I'm not at home (since blocking the sites has no effect if I'm not on my wifi network).

However, my router's stock firmware does not provide SSH access, and the web-based admin portal is 100% javascript, which prevents me from scripting it with an auto-browser library.

So, in order to allow the RPi to give instructions to the router, I installed the LEDE firmware, and set up its SSH server.

#### Am I at home?

The next step was to set up a way for my RPi to check if my iPhone is on the network -- that can be accomplished by checking if my iPhone's MAC address is registered with the router:

```bash
iw dev wlan0 station dump | grep -q $IPHONE_MAC
```

This command needs to be run on the router itself, which we can do over SSH:

```bash
sshpass -p $ROUTER_SSH_PASS ssh root@192.168.1.1 "iw dev wlan0 station dump" | grep -q $IPHONE_MAC
```

So, we can write the below bash script to check if I'm at home and run the process accordingly:

```bash
if sshpass -p $ROUTER_SSH_PASS ssh root@192.168.1.1 "iw dev wlan0 station dump" | grep -q $IPHONE_MAC; then
	echo "iPhone is on the network, checking workout status..."
	python /path/to/workout.py
else
	echo "iPhone is not on the network, exiting"
fi
```

#### Set up website blocking

Next, we need to set up the ability to block and unblock certain domains.  The LEDE router firmware has a library called `simple-adblock` that blocks domains associated with serving ads — we can piggyback on this library to block our distracting domains as well.

Once `simple-adblock` is installed, it puts a domain block list at `/etc/config/simple-adblock` (may vary based on your setup).  Reading through that file, the syntax for blocking a domain becomes apparent:

```bash
list blacklist_domain $DOMAIN
```

Therefore, to block facebook, we would append the following line to the file:

```bash
list blacklist_domain 'facebook.com'
```

I created two versions of the block list in the router's home directory — one including the domains I want to block, and the other excluding those domains.  Our script will copy the right list over the existing `simple-adblock` configuration depending on the output from the MFP monitoring script.

The script first compares (`cmp`) the relevant block list to the existing configuration — if it matches, there is no need to change the configuration, and we can exit.  If it doesn't match, we need to copy over the new list and execute a few commands to have the router reset its blocking:

```bash
if cmp -s /path/to/block-list /etc/config/simple-adblock
then
    echo "Already blocking web access"
else
    cp /path/to/block-list /etc/config/simple-adblock
    /etc/init.d/simple-adblock reload
    /etc/init.d/dnsmasq restart
    echo "Web access blocked"
fi
```

And, therefore, to unblock:

```bash
if cmp -s /path/to/unblock-list /etc/config/simple-adblock
then
    echo "Already blocking web access"
else
    cp /path/to/unblock-list /etc/config/simple-adblock
    /etc/init.d/simple-adblock reload
    /etc/init.d/dnsmasq restart
    echo "Web access blocked"
fi
```

#### Connect router interaction to MFP monitoring

Above, in the MFP python script, we included two functions, `block_sites` and `unblock_sites` that need to interact with the router.  Since we wrote the block and unblock bash scripts above (which live on the router), we can go ahead and implement these functions on the RPi:

```python
def block_sites():
	os.system('sshpass -p $ROUTER_PASS ssh -T root@192.168.1.1 "/path/to/router/block/script"')
    
def unblock_sites():
	os.system('sshpass -p $ROUTER_PASS ssh -T root@192.168.1.1 "/path/to/router/unblock/script"')
```

### Automating the process

We have basically set up the entire process to follow an execution chain:

RPi script to check if home -> RPi script to determine if I've exercised -> Router script to block/unblock sites

So, the only script we need to call to run the process is the RPi bash script that checks if I'm home (and runs the MFP script if needed).

We can do that easily by adding the bash script to the RPi's crontab.  I have my RPi set up to run the script every fifteen minutes:

```bash
*/15 * * * * bash /path/to/workout.sh
```

### Reference - Full Scripts

For reference, the full scripts, incorporating all the above pieces, are below:

```python
#workout.py - python script that is run on RPi
import myfitnesspal
import datetime
import os

mfp = myfitnesspal.Client($MFP_EMAIL)

def has_exercised(date):
	has_exercised = False
	mfp_exercise = mfp.get_exercise(date.year, date.month, date.day)

	cardio = mfp_exercise[0]
	for cardio_exercise in cardio:
		if cardio_exercise.name != 'MFP iOS calorie adjustment':
			has_exercised = True

	strength = mfp_exercise[1]
	if len(strength) > 0:
		has_exercised = True

	return has_exercised

def block_sites():
	os.system('sshpass -p $ROUTER_PASS ssh -T root@192.168.1.1 "/path/to/router/block/script"')
    
def unblock_sites():
	os.system('sshpass -p $ROUTER_PASS ssh -T root@192.168.1.1 "/path/to/router/unblock/script"')

today = datetime.date.today()
if not(has_exercised(today)):
	print "No exercise detected, blocking web access..."
	block_sites() 
else:
	print "Exercise detected, unblocking web access..."
	unblock_sites()
```

```bash
#workout.sh - bash script to check if home, and run python MFP script
if sshpass -p $ROUTER_SSH_PASS ssh root@192.168.1.1 "iw dev wlan0 station dump" | grep -q $IPHONE_MAC; then
	echo "iPhone is on the network, checking workout status..."
	python /path/to/workout.py
else
	echo "iPhone is not on the network, exiting"
fi
```

```bash
#block-sites.sh - router script to block sites
if cmp -s /path/to/block-list /etc/config/simple-adblock
then
    echo "Already blocking web access"
else
    cp /path/to/block-list /etc/config/simple-adblock
    /etc/init.d/simple-adblock reload
    /etc/init.d/dnsmasq restart
    echo "Web access blocked"
fi
```

```bash
#unblock-sites.sh - router script to unblock sites
if cmp -s /path/to/unblock-list /etc/config/simple-adblock
then
    echo "Already blocking web access"
else
    cp /path/to/unblock-list /etc/config/simple-adblock
    /etc/init.d/simple-adblock reload
    /etc/init.d/dnsmasq restart
    echo "Web access blocked"
fi
```

## Potential Enhancements

As written, this is a pretty simple workflow, as it blocks/unblocks sites upon the presence of a workout.  With the data available to us in MFP and the capability of the router, there are a few enhancements that could be made:

- Require burning a set number of calories to unblock sites (e.g. need to burn 400 calories before unblocking)
- Require exercising for a certain length of time to unblock (e.g. need 30 minutes of cardio)
- Add ability to determine the amount of time allowed on blocked sites by the amount of exercicse completed (e.g. 30 mins of exercise buys you 30 mins of internet access)
- Block internet access if you exceed your calorie intake or net calorie goal for the day
- Block internet access if you eat a certain type of food (e.g. block sites if candy is logged to MFP)
- Require a certain number of steps to unblock

In addition to MFP, the RPi can use any other data source as an input to the block/unblock algorithm.  For example, the RPi could be linked to an app like RescueTime to block sites until the user has completed a certain amount of work time, or to block sites if the user hits a certain amount of 'distracted time'.

## Comments / Forks

Feel free to fork this and apply it as you see fit, and to leave comments or suggestions for improvement.