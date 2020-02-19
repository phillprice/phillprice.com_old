---
title: "Home Assistant Google Calendar Actionable Notifications"
date: 2020-02-19T20:15:46Z
draft: false
description: "HOw I get notified its time to take out the bins."
featured_image: "bin-notifications.png"
categories: ["Home Assistant"]
---

I was toying with how to remind myself to put the right bin out at the right time. Although its generally the same schedule every week, I would still occasionally forget.

I used to be able to html scrape a local council page for the next date, but they changed supplier and that’s now gone :frowning:

Instead I created a new Google Calendar for each bin (and made the colour match the bin, naturally…)

To do so click the + next to calendar, choose New Calendar and give it a Name and Colour

Follow the guide here on how to get Google Calendars into HA 60. You can choose to track or not track all the other calendars it brings in. I use this for lots of things from guest mode to travel sensors, wfh modes and other bits and bobs. Its quite powerful.

Once you’ve got the calendars into HA as sensors (calendar.blue_bin, calendar.black_bin etc) you can then build up on it.

To start with you’ll probably need to make recurring all day appointments in Google. If like me you don’t want to use the powerful tagging to create offsets as outlined in the HA docs you’ll want a trigger sensor to turn on some time the evening before.

I do this by making two template sensors: (25200 seconds is 7 hours, so the sensor triggers at 5pm)

```yaml
- platform: template
  sensors:
    blue_bin_offset:
      friendly_name: "blue bin offset"
      entity_id: binary_sensor.bins_offset_trigger
      value_template: >
        {% if as_timestamp(states.calendar.blue_bin.attributes.start_time) - as_timestamp( strptime(states.sensor.date__time.state, "%Y-%m-%d, %H:%M" ) ) < 25200  %}on{% else %}off{% endif %}
- platform: template
  sensors:
    green_bin_offset:
      friendly_name: "green bin offset"
      entity_id: binary_sensor.bins_offset_trigger
      value_template: >
        {% if as_timestamp(states.calendar.green_bin.attributes.start_time) - as_timestamp( strptime(states.sensor.date__time.state, "%Y-%m-%d, %H:%M" ) ) < 25200  %}on{% else %}off{% endif %}
```
We’ll also need an input_boolean to stop notifying when we’ve actually put the things out. We’ll turn it on later.

```yaml
bins_out:
  name: Bins Out
  icon: mdi:delete-empty
```
EDIT: We also need a time of day sensor to trigger updating the offset sensors or they will stay dormant.

```yaml
- platform: tod
  name: Bins Offset Trigger
  after: '17:00'
  before: '20:00'
```
Now we have the triggers we need automations to kick off to notify me. They check if the bins are already out (maybe i remembered earlier, or maybe between reminders?) Note in the iOS notification action the data -> push we’ll set that up shortly.

```yaml
- id: '1529959662644'
  alias: Green Bin Reminder
  trigger:
  - entity_id: sensor.green_bin_offset
    platform: state
    to: 'on'
  condition:
  - condition: state
    entity_id: input_boolean.bins_out
    state: 'off'
  action:
  - data:
      message: The GREEN Bin needs to go out
      title: Bin Day!
      data:
        push:
          category: greenbins
    service: notify.ios_phill
- id: '1529959662643'
  alias: Blue Bin Reminder
  trigger:
  - entity_id: sensor.blue_bin_offset
    platform: state
    to: 'on'
  condition:
  - condition: state
    entity_id: input_boolean.bins_out
    state: 'off'
  action:
  - data:
      message: The BLUE Bin needs to go out
      title: Bin Day!
      data:
        push:
          category: bluebins
    service: notify.ios_phill
```
OK so the actions bit, the greenbins and bluebins category. We’ll need to set that up. I don’t really understand the bit below title though… the identifier is the event pushed back to HA, the title is the label in the notification when you force push it to see more detail.

```yaml
ios:
  push:
    categories:
      - name: Blue Bins
        identifier: 'bluebins'
        actions:
          - identifier: 'BINS_OUT'
            title: 'Bins are out'
            activationMode: 'background'
            authenticationRequired: yes
            destructive: yes
            behavior: 'default'
          - identifier: 'BLUE_REMIND_LATER'
            title: 'Remind me later'
            activationMode: 'background'
            authenticationRequired: yes
            destructive: no
      - name: Green Bins
        identifier: 'greenbins'
        actions:
          - identifier: 'BINS_OUT'
            title: 'Bins are out'
            activationMode: 'background'
            authenticationRequired: yes
            destructive: yes
            behavior: 'default'
          - identifier: 'GREEN_REMIND_LATER'
            title: 'Remind me later'
            activationMode: 'background'
            authenticationRequired: yes
            destructive: no
```
When one of the actions is tapped we’ll need automations to pick that up and do stuff

One to change the input_boolean.bins_out Note is sends an acknowledgement back for peace of mind.

```yaml
- id: '1529960481866'
  alias: Bins are out boolean changer
  trigger:
  - event_data:
      actionName: BINS_OUT
    event_type: ios.notification_action_fired
    platform: event
  condition: []
  action:
  - data:
      entity_id: input_boolean.bins_out
    service: input_boolean.turn_on
  - data:
      message: Bin notification received.
    service: notify.ios_phill
```
And we’ll need the remind later automations which kick of scripts to delay kicking off the reminder notifications manually (instead of the sensor triggering them)

```yaml
- id: '1529960481868'
  alias: Blue Bin Remind in an hour
  trigger:
  - event_data:
      actionName: BLUE_REMIND_LATER
    event_type: ios.notification_action_fired
    platform: event
  condition: []
  action:
  - data:
      entity_id: script.blue_bin_in_an_hour
    service: script.turn_on
  - data:
      message: OK I will remind you in an hour.
    service: notify.ios_phill
- id: '1529960481869'
  alias: Green Bin Remind in an hour
  trigger:
  - event_data:
      actionName: GREEN_REMIND_LATER
    event_type: ios.notification_action_fired
    platform: event
  condition: []
  action:
  - data:
      entity_id: script.green_bin_in_an_hour
    service: script.turn_on
  - data:
      message: OK I will remind you in an hour.
    service: notify.ios_phill
```
Here are the basic scripts that do those delays:

```yaml
blue_bin_in_an_hour:
  alias: Remind about Blue Bins in an hour
  sequence:
  - delay: 01:00
  - data:
      entity_id: automation.blue_bin_reminder
    service: automation.trigger
green_bin_in_an_hour:
  alias: Remind about Green Bins in an hour
  sequence:
  - delay: 01:00
  - data:
      entity_id: automation.green_bin_reminder
    service: automation.trigger
```
Now all that’s needed is a second trigger on the day in case i just ignore the reminder (although i could nag by simply starting the script again each time it finishes?) Note the 5am script means it triggers an hour later at 6am.

```yaml
- alias: Remind green bin at 6am
  trigger:
  - at: 05:00
    platform: time
  condition:
  - condition: state
    entity_id: input_boolean.bins_out
    state: 'off'
  - condition: state
    entity_id: calendar.green_bin
    state: 'on'
  action:
  - data:
      entity_id: script.green_bin_in_an_hour
    service: script.turn_on
  id: d6581bd23a8e4c01a60428b84cdb7985
- alias: Remind blue bin at 6am
  trigger:
  - at: 05:00
    platform: time
  condition:
  - condition: state
    entity_id: input_boolean.bins_out
    state: 'off'
  - condition: state
    entity_id: calendar.blue_bin
    state: 'on'
  action:
  - data:
      entity_id: script.blue_bin_in_an_hour
    service: script.turn_on
  id: d6581bd23a8e4c01a60428b84cdb7986
```
How can this be improved? Well it would be nice to automate the bins_out boolean by possibly using beacon locations or machine learning camera images but the bins get moved around. I worry the beacons woudl get knocked off or the bins swapped.

Any other improvement ideas? Steps I’ve missed? Shout!

EDIT: Forgot how the boolean gets turned off.

Firstly there is a group of bins

```yaml
bin_group:
   entities:
    - calendar.blue_bin
    - calendar.green_bin
Then there’s an automation on it turning off
```

```yaml
- id: '1530193864185'
  alias: Bins in when Bins in
  trigger:
  - entity_id: group.bin_group
    platform: state
    to: 'off'
  condition: []
  action:
  - data:
      entity_id: input_boolean.bins_out
    service: input_boolean.turn_off
```