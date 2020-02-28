---
title: "How I know I'll be late with NodeRed and Home Assistant"
date: 2020-02-24T19:00:00Z
draft: false
categories: ["Home Assistant"]
---

If, like me, you use the rail network to get to work you'll understand how annoying it is when I turn up to the station and see a big group of people and no trains.

To get around this I use the [UK Transport API](https://developer.transportapi.com/), through Node Red and MQTT, to make a dashboard in [Home Assistant](https://www.home-assistant.io/getting-started/) and notify my phone if there are more than 3 delayed trains and the delays are more than 5 minutes on average. (Don't worry this tolerance is configurable later on).

This guide will help you, through the installation of a few [Home Assistant](https://www.home-assistant.io/getting-started/) Add-ons and minimal editing hopefully achieve this high state of zen.

## Prerequisites

* You use UK rail (or bus!) network to get to work
* [Home Assistant](https://www.home-assistant.io/getting-started/), [Node Red](https://github.com/hassio-addons/addon-node-red) and [MQTT](https://github.com/home-assistant/hassio-addons/blob/master/mosquitto/README.md) working together.
* A geolocating device tracker and [zones](https://www.home-assistant.io/blog/2020/02/05/release-105/#improved-zones-editor) set up for the stations, _or you'll get notifications about trains to work when you're already there!_
* An existing Notification service. I use telegram as it has actionable notifications. You may need to tailor some nodes discussed later for it to work for you.

## API Setup
The first thing we'll need to do is set up an API account at transport API. To do so go to their [developer site](https://developer.transportapi.com/) and Sign up. We'll need to keep the `App ID` and `Key` for requests later on.

In this guide I'll be using my commute, from `Woking` to `London Waterloo` and back. These stations are used in the API and in our supporting nodes and home assistant configuration under their `CRS` (Computer Reservation System) codes `WOK` and `WAT` respectively. Please replace them using your station codes. They can be found in the [railway codes](http://www.railwaycodes.org.uk/crs/CRS0.shtm) website.

> [In the API request] Stations can be referenced in the form `{scheme}:{code}`:

> `crs:wat` references the `CRS` code `WAT` for Waterloo station.
> `tiploc:watrloo` references the `TIPLOC` `WATRLOO` for Waterloo station.
> if no scheme is specified, `CRS` is assumed by default, so `wat` implicitly references the `CRS` code `WAT` for Waterloo station.

## Home Assistant Setup
We'll need to set up some items within Home assistant to track the state of what's happening and for display in the lovelace interface.

### Input Booleans for Notification Suppression
First we set up some input booleans to allow us to suppress notifying us through actionable notifications.

{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "suppression_booleans.yaml" >}}

### Input Booleans for Zone entry
We'll also add some input booleans that we'll toggle through a `device_tracker` when we arrive at stations to suppress notifications when we're already on a train.

{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "location_booleans.yaml" >}}

### MQTT sensors for saving state
We'll also need to set up a couple of mqtt sensors, which take the state as the number of late trains and a whole lot of information as attributes. Mine uses the CRS codes again for the station as part of the name.

{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "mqtt_sensors.yaml" >}}

## Node-Red setup
Now that we have these booleans and sensors setup We can start polling the API for data. Note that the Transport API has a __1000 request limit per day__. This equates to one call every 87 seconds. In this case we have four requests. Our interval will need to be a minimum of 348 seconds. I've rounded that up to 6 minutes in the below flow.

_Note that unless we have 24/7 trains and a 24/7 need for updates we can limit the window it makes requests in the `timestamp` node and therefore increase frequency! See timestamp injection below._

The full flow is here. Let us break it up into sections to see what it does.

{{< srcset src="flow.png" title="Full Node Red Flow">}}

{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "flow.json" >}}

### Part 1. Getting the data from the API
{{< srcset src="step_1.png" title="Notifications">}}
#### Timestamp Injection
The top left node is where we start the flow activity. Earlier I noted that we can run the flow every 6 minutes, but if we know we're not going to need train information early in the morning, late at night or during weekends we can amend this node's details to turn off days and limit the hours the API is polled. This allows us to increase frequency.

{{< srcset src="injection.png" title="Injection node" width="445" >}}

#### Train Details
In the next nodes we set the train specific payload, each of these items get used in the API request. We ask for arrivals and departures because it can easily be the case that problems occur further along the train's journey and the departures may not be experiencing them yet.

{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "train_departures.js" >}}

Here's an arrival node:

{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "train_arrivals.js" >}}


#### API Details

Next we set up our API ID and Key from Transport API. We look for arrival and departures slightly differently.

 _Note we'll need to add the details into a node in Part 2, where the API is polled again, I'm sure there is another way to set this more globally? - answers in the comments!_

{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "apidetails.js" >}}

#### Handling Returned Data
After the HTTP request itself, we handle the data, squishing it into what we need as state and attributes. This node is full of information and the whole JS is here:

{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "uktransportapi.js" >}}

Let us look a bit more about what happened there:
* We get the list of trains and reverse it (to help remove overtaken trains as we go backwards on arrival time. If the arrival time is later than the train after it we remove it...)
* For each train we use `time_check` to work out which trains to remove, and we work out the delay and duration, added them to the train object.
* After removing overtaken trains we work out the number of delayed trains (we'll use that as the state), the average delay and the maximum delay
* We then work out a list of each status and the number of trains with that status using javascript filters.
* Just before ending we'll duplicate the attributes into a payload item called telegram, this way the payload key is predictable with different trains before sending that down to Part 2.
* Finally we work out which MQTT topics to post to and send it to MQTT

### Part 2. Delay notifications
{{< srcset src="step_2.png" title="Notifications">}}
* We first move the payload to a different attribute to make sure it is not changed.
* We then work out which train it is by looking at the `from` attribute
* We check to see if notifications for this line are suppressed (see part 3)
* We check to see if we've already passed through the station (see part 4)
* We then push the data back into the payload ready for the meat of the notification in the following gist:
{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "notification.js" >}}
  - The main things to worry about here is line 5 where we set the tolerance to ignore less than 3 late trains and line 11 the tolerance for how delayed they must be before a notification.
  - Lots of this node is some lovely work into moving things into different variables and adding 's' to plurals.
  - However, there is a second line in the notification that says `5 Cancelled` etc. We look through the attributes for a whitelist of problem statuses and their sums for that.
  - Finally we build the payload string, the message itself and set up the actionable notifications for part 3

{{< srcset src="notification.jpg" title="Notifications">}}

### Part 3. Suppressing unwanted notifications
{{< srcset src="step_3.png" title="Notifications">}}
Other than the overall tolerance for trains
Telegram (and other notification methods but I don't know them, sorry) can reply with buttons for messages. In this flow we have three notifications we can reply for each train:
* TRAINS WOK WAT SHOW
* TRAINS WOK WAT 30
* TRAINS WOK WAT DAY

The first one triggers as if we entered a station, we'll look at that in Part 4.
The other two callbacks turn on the suppression boolean. For the `30` it then sets a timer to turn it off again. For `DAY` we rely on Part 5 where an overnight rigger turns them off.

### Part 4. Location-based notifications
{{< srcset src="step_4.png" title="Notifications">}}
* Using our device tracker and setting up a zone for each station we check if we've entered a location. We can also trigger this from the delay notification in Part 2 and Part 3.
* We check to see if its morning or not and then pass a pre-baked API request
* We then turn that payload into a notification like this:
{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "locationnotification.js" >}}
  - Most of this is simple string manipulation.
  - If there is a delay we add that to the line:

    `Platform 2 08:32 to London Waterloo`

    `Platform 2 08:32 to London Waterloo (exp 08:47)`


### Part 5. Resetting overnight.
{{< srcset src="step_5.png" title="Notifications">}}
Now that we're turning on booleans when we enter locations and when we suppress notifications we'll need to turn them off to reset for the next day. The last two nodes on the flow do exactly this

## Lovelace setup
{{< srcset src="lovelace.png" title="Injection node" width="445" >}}
You're probably wondering how I setup the lovely header image.
We'll need the [dual guage card](https://github.com/custom-cards/dual-gauge-card) and the [HTML Jinja2 Template card](https://github.com/PiotrMachowski/Home-Assistant-Lovelace-HTML-Jinja2-Template-card),  which are installed through [HACS](https://github.com/hacs/integration).

Then add this view:
{{< gist phillprice 47f3945ff9bdc300624f4dcd6aa43df6 "lovelace.yaml" >}}


## TLDR; Things we'll need to change
So, in summary, this is all of the detail we'll need to change, I think, to get this working:
* Start and end stations
* Any references to sensors and booleans we've amended
* Telegram Chat ID (_or the payload setup/callback for other notification services_)
* Transport API ID and Key
* Tolerances to silence alarms
* Device Tracker name
* Zone Names from the device trigger.

## Extra Trains
{{< srcset src="myflow.png" title="Notifications">}}
In my real life I change trains and I therefore I have four, not two, of everything. I also have another set of a different route for my wife setup. You can see above that its a lot of nodes, but it is not huge effort.

## Caveat
It goes without saying that this all goes fine until the data isn't correct. During the write up of this guide I turned up and there were no trains but the train company didn't update the service status and the TransportApi was happily telling me everything was fine because that's what they were being told. Its a rarity though.