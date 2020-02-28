---
title: "Home Assistant, Google Calendar Actionable Notifications"
date: 2020-02-19T20:15:46Z
draft: false
description: "How I get notified its time to take out the bins."
categories: ["Home Assistant"]
aliases:
    - /blog/home-assistant-google-calendar-actionable-notifications/index.php
---

I was toying with how to remind myself to put the right bin out at the right time. Although its generally the same schedule every week, I would still occasionally forget.

I used to be able to html scrape a local council page for the next date, but they changed supplier and that’s now gone   :frowning:

Instead I created a new Google Calendar for each bin (and made the colour match the bin, naturally…)

## Google Calendar Setup

In Google Calendar on the left click the `+` next to calendar, choose `New Calendar` and give it a Name and Colour

[Follow the guide here](https://www.home-assistant.io/integrations/calendar.google/) on how to get Google Calendars into HA. You can choose to track or not track all the other calendars it brings in. I use this for lots of things from guest mode to travel sensors, wfh modes and other bits and bobs. Its quite powerful.

Once you’ve got the calendars into HA as sensors (calendar.blue_bin, calendar.black_bin etc) you can then build up on it.

## Event Setup

To start with you’ll probably need to make recurring all day appointments in Google. If like me you don’t want to use the powerful tagging to create offsets as outlined in the HA docs you’ll want a trigger sensor to turn on some time the evening before.

## Home Assistant Setup
I do this by making two template sensors: (25200 seconds is 7 hours, so the sensor triggers at 5pm)

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "offset.yaml" >}}
We’ll also need an input_boolean to stop notifying when we’ve actually put the things out. We’ll turn it on later.

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "bins_out.yaml" >}}
We also need a time of day sensor to trigger updating the offset sensors or they will stay dormant.

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "tod_trigger.yaml" >}}
## Actionable Notification Automations
{{< srcset src="notification_iphone.png" title="Full Node Red Flow">}}
Now we have the triggers we need automations to kick off to notify me. They check if the bins are already out (maybe i remembered earlier, or maybe between reminders?) Note in the iOS notification action the data -> push we’ll set that up shortly.

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "notification.yaml.yaml" >}}
OK so the actions bit, the greenbins and bluebins category. We’ll need to set that up. I don’t really understand the bit below title though… the identifier is the event pushed back to HA, the title is the label in the notification when you force push it to see more detail.

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "ios.yaml.yaml" >}}
When one of the actions is tapped we’ll need automations to pick that up and do stuff

One to change the input_boolean.bins_out Note is sends an acknowledgement back for peace of mind.

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "turn_on_bins.yaml.yaml" >}}
And we’ll need the remind later automations which kick of scripts to delay kicking off the reminder notifications manually (instead of the sensor triggering them)

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "bin_reminder.yaml.yaml" >}}
Here are the basic scripts that do those delays:

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "script.yaml.yaml" >}}
Now all that’s needed is a second trigger on the day in case i just ignore the reminder (although i could nag by simply starting the script again each time it finishes?) Note the 5am script means it triggers an hour later at 6am.

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "early_bin_reminder.yaml.yaml" >}}
How can this be improved? Well it would be nice to automate the bins_out boolean by possibly using beacon locations or machine learning camera images but the bins get moved around. I worry the beacons would get knocked off or the bins swapped.

Any other improvement ideas? Steps I’ve missed? Shout!

## Resetting the booleans after bin day

Firstly there is a group of bins

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "group.yaml.yaml" >}}
Then there’s an automation on it turning off

{{< gist phillprice 90a2c828baa98f4c01c278090a5de7f8 "off_automation.yaml.yaml" >}}

Hopefully that all made sense?