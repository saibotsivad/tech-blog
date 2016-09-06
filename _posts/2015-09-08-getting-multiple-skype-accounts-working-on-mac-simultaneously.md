---
title: Multiple Skype Accounts on OS X
date: Tue September 9 2015
author: Randy D. Wallace Jr.
layout: post
---

It's been asked [many times](https://www.google.com/search?q=multiple%20skype%20accounts%20mac), but I was never quite satisfied with the answer, so I took what was out there and improved upon it a bit to serve my needs.

This solution will provide the illusion of multiple installations of Skype that you can run under one user account.  There is no `sudo` here.

## Preparation

You're going to follow the following steps for as many Skype accounts as you need.  If you're here, you probably need to have a Personal and Work Skype account.  These steps document what is necesary to
have one Custom Skype shortcut.  You will need to follow these steps for as many Skype accounts you need to be logged into.

This works by using an undocumented feature Skype provides that allows you to tell it upon startup what folder to use for its data.  I don't take full credit for this, refer to 
this [Superuser post](http://superuser.com/questions/271646/multiple-skype-clients-on-mac-os-x) for a little more.  All I've added is the Automator flair that makes this user friendly.

## Step 1

1. Track down `Automator`.  You can find this application either buried in the `Other` Folder in the Launchpad or via Spotlight.

![Automator Image](/content/assets/skype-screenshot-automator.png)

2. When asked to `Choose a type for your document`, select `Application`.

## Step 2

1. Go ahead and save this application to your OS X Account's `Applications` folder; note that this is not the primary Applications folder which shows up by default in the Favorites in Finder, but the Applications folder that is 
inside your User account's directory (i.e. /Users/<your_user_account_name>/Applications/ or the folder with a Home icon in Finder).  Give it the name you want to see in Launchpad, i.e. `Skype Personal`.
2. At this point, when you pull up the Launchpad, you should see your application there (although it doesn't actually do anything yet).

## Step 3

1. In Automator, track down the `Run Shell Script` Action and drag it over to the right.
2. In the `Shell` dropdown, select `/bin/bash`.
3. Inside the text entry box, copy and paste the text below, replacing `Personal` with whatever you would like.
```
CUSTOM_SKYPE_DATA_FOLDER_NAME="Personal"

DATA_PATH="/Users/$(whoami)/Library/Application Support/Skype ${CUSTOM_SKYPE_DATA_FOLDER_NAME}"

mkdir -p "${DATA_PATH}"

nohup open -na /Applications/Skype.app --args -DataPath "${DATA_PATH}" > /dev/null 2>&1 &

exit 0
```
4. At the bottom of the action window for `Run Shell Script` is an `Options` button.  Select that and check the box labeled `Ignore this action's input`.  It should look like this when you're done:

![Run Shell Script Image](/content/assets/skype-screenshot-run-shell-script.png)

5. Save!


## Step 4

1. Go ahead and test it from the Launchpad!  It should open Skype.  Notice that it behaves as if it has never been logged into before.  Great!  Go ahead and login with the account you want this particular
shortcut to use.  In this case, I logged into my personal Skype account.
2. You can stop here if you're happy and go ahead and create other shortcuts, or, keep going to Step 5 to class it up a bit.

## Step 5 (optional)

In this step, you're going to copy the Skype Icon from the Primary Skype application to your new Automator Application.  Also, you can refer to this 
[Macworld Article](http://www.macworld.co.uk/how-to/mac-software/how-change-os-x-yosemites-icons-3597494/) for more elaborate instructions (with screenshots).

1. Open up the System Applications folder in Finder.
2. Right click on the Skype Application Icon and select `Get Info`.
3. Ok, this is gonna sound crazy if you've never heard of this, but at the very top left of that window, click *directly* upon the Skype logo.  You should see the icon change (it will have a border).  At this
point, hit copy (Edit => Copy or CMD+C).
4. Now, go track down your Automator Application in your User account's Applications folder, right click on it, and open up that application's `Get Info`.
5. Now, click *that* application icon in the top left (the automator) until it changes, *then* hit paste.  You should see the Automator turn into a Skype logo.

## Finishing up

![Final Image](/content/assets/skype-screenshot-final.png)

Well, that about wraps it up.  This has worked great for me, and I hope it does for you too.  Just create as many of these Automator Applications as you need, each with different `CUSTOM_SKYPE_DATA_FOLDER_NAME`'s and File Names, and 
you should be able to run them all simultaneously.  Keep an eye on your resources though, I don't know what the fallout is from running more than 2 copies of Skype simultaneously!
