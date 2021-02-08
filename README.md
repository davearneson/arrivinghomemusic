# Arriving Home -  Play to Alexa Multi Room Audio Setup

![Arriving Home - Whole Home Alexa Audio](https://i.imgur.com/cuPjrpG.png)

This is building off of [this previous project of mine](https://github.com/davearneson/alexawholehome) to create a system of automations to automatically play a variety of music of my choosing on my Alexa Multi-Room Music Groups when I get home from work.

## Alexa Media Player Setup

This project uses the [Alexa Media Player](https://github.com/custom-components/alexa_media_player) custom component to interact with my Alexa Devices. I won't go into detail on setting this up, because I haven't deviated any from the standard setup for this custom component.

As I mentioned in the previous project, [there is currently a limitation to Alexa Media Player](https://github.com/custom-components/alexa_media_player/wiki#play-in-alexa-groups) which means you can start and stop music to individual echo devices, but you can't send music directly to an Alexa Multi-Room Music Group. The workaround that I've found to be most consistent - which I use in this project - is sending a custom simulated voice command to Alexa through the component, like "Play Jazz on the Downstairs Music Group."

## Home Assistant Helpers

I used HA’s helpers to set up an input_text entity in the UI by going to Configuration/Helpers and clicking the add button, then “Text.” I then added a name, in my case “Arriving Home Music”. This entity will be used to type text for whatever music or audio I want sent to the Alexa Music Groups. I changed the icon to "mdi:music"

![Helper Setup for Arriving Home - Whole Home Alexa Audio](https://i.imgur.com/hDv1B0q.png)
# Actionable Notification
I have an iPhone, so this part of the uses actionable notifications for iOS. The process to set this up is well documented here: https://companion.home-assistant.io/docs/notifications/actionable-notifications/

![Arriving Home Music Actionable Notification](https://i.imgur.com/QmAIbKf.jpg)

## configuration.yaml

I added one section to configuration.yaml for Actionable Notifications for this project:

```
# configuration.yaml

ios:
  push:
    categories:
      - name: Todays Music
        identifier: 'todaysmusic'
        actions:
          - identifier: 'INPUT_MUSIC'
            title: "Type what you'd like to hear"
            behavior: 'textInput'
            textInputButtonTitle: 'Submit'
            textInputPlaceholder: 'music'
          - identifier: 'JAZZ_MUSIC'
            title: 'Jazz'
          - identifier: 'Classical_MUSIC'
            title: 'Classical'
          - identifier: 'RANDOM_MUSIC'
            title: 'Random'
```
The "textInput" behavior allows me to respond inside the actionable notification with text that will be sent to Home Assistant:

![Arriving Home Music Actionable Notification](https://i.imgur.com/ZlgSEsK.jpg)
# Automations

I created three automations for this project, and I’ll do my best to explain them and the logic behind them:

## What Music Do You Want? Notification

The first few automation sends the actionable notification at prescribed time, about 15 minutes before I leave work each weekday.

```
# automations.yaml

- alias: Arriving Home What Music?
  description: ''
  trigger:
  - platform: time
    at: '16:15:00'
  condition:
  - condition: time
    weekday:
    - mon
    - tue
    - wed
    - thu
    - fri
  - condition: zone
    entity_id: person.me
    zone: zone.my_work
  action:
  - data:
      data:
        push:
          category: todaysmusic
      message: What do you want to hear today?
      title: Arriving Home Music
    service: notify.mobile_app_my_iphone
  mode: single
```
![Arriving Home Music Actionable Notification](https://i.imgur.com/ZMAB6TL.jpg)
## Automation Components

This automation to process the return from the Actionable Notification is where it gets fun. I set the triggers for this one up in the UI to make things easier - there are triggers for each of the actionable notification returns I set up earlier (JAZZ_MUSIC, CLASSICAL_MUSIC,  RANDOM_MUSIC, and INPUT_MUSIC). Then I set up the action to fire a service, using as an if statement as the data value (I had to set this up in YAML, because it uses "data_template"). This is using the input_text.set_value service to change the helper I created earlier to whatever my response from the actionable notification was.

```
# automations.yaml

- alias: Arriving Home What Music? - Component to Set Text Input
  description: ''
  trigger:
  - platform: event
    event_type: ios.notification_action_fired
    event_data:
      actionName: JAZZ_MUSIC
    context:
      user_id:
      - ****************************
  - platform: event
    event_type: ios.notification_action_fired
    event_data:
      actionName: CLASSICAL_MUSIC
    context:
      user_id:
      - ****************************
  - platform: event
    event_type: ios.notification_action_fired
    event_data:
      actionName: INPUT_MUSIC
    context:
      user_id:
      - ****************************
  - platform: event
    event_type: ios.notification_action_fired
    event_data:
      actionName: RANDOM_MUSIC
    context:
      user_id:
      - ****************************
  condition: []
  action:
  - data_template:
	  entity_id: input_text.arriving_home_music
	  value: >-
	    {% if trigger.event.data["actionName"] == "JAZZ_MUSIC" %}
	      Jazz Radio on Apple Music
	    {% elif trigger.event.data["actionName"] == "CLASSICAL_MUSIC" %}
	      Classical Radio on Apple Music
	    {% elif trigger.event.data["actionName"] == "INPUT_MUSIC" %}
	      {{ trigger.event.data["textInput"] }}
	    {% else %}
	      Music
	    {% endif %}
	service: input_text.set_value
  mode: single
```
## Play Music Automation
Finally, this last automation is what sets off the actual music when I get home from work. It's simply using HA zones and my iOS device to see when I enter the "home" zone. I have it time-conditioned, and conditioned for when my spouse is not already home, because they would probably not appreciate being blasted with my music if they're home watching TV or just generally minding their own business.

```
# automations.yaml

- alias: Arriving Home Play Music
  description: ''
  trigger:
  - platform: zone
    entity_id: person.me
    zone: zone.home
    event: enter
  condition:
  - condition: time
    after: '12:00:00'
    before: '21:30:00'
  - condition: not
    conditions:
    - condition: state
      entity_id: person.spouse
      state: home
  action:
  - service: media_player.play_media
    data_template:
      entity_id: media_player.echo_link
      media_content_id: play {{ states('input_text.arriving_home_music') }} on
        the downstairs group
      media_content_type: custom
```

# Scripts
I also created one script for use in this project. This is an (overkill) fail-safe to make sure all music and audio stops if I want it to stop. I made this as a script so I could control it through automations and manual input buttons in the UI.
```
##############################################################################        
#################Script to stop all audio on all groups#######################
##############################################################################
stop_alexa_music:
  alias: Stop Music on Alexa Devices
  sequence:
    - service: media_player.play_media
      data:
        entity_id: media_player.echo_link
        media_content_id: stop music on everywhere group
        media_content_type: custom
    - service: media_player.play_media
      data:
        entity_id: media_player.echo_link
        media_content_id: stop music on front of house group
        media_content_type: custom
    - service: media_player.play_media
      data:
        entity_id: media_player.echo_link
        media_content_id: stop music on downstairs group
        media_content_type: custom
    - service: media_player.play_media
      data:
        entity_id: media_player.echo_link
        media_content_id: stop music on upstairs group
        media_content_type: custom
    - service: media_player.play_media
      data:
        entity_id: media_player.echo_link
        media_content_id: stop music on master bedroom group
        media_content_type: custom
    - service: media_player.play_media
      data:
        entity_id: media_player.echo_link
        media_content_id: stop music
        media_content_type: custom
   ```

![Stop Music as a button in Lovelace](https://i.imgur.com/wuJB5lk.png)
This button is just sending "stop music" commands to all possible groups.
# Final Thoughts
I went back and forth on whether I wanted to create another automation to reset the music choice each day, or if I wanted to be able to leave it on a selection for as long as I feel like it. For now, I'm sticking with letting the previous day's choice hold over until I decide to change it. I've had this set up for about a week so far, and overall, I'm really happy with the setup.
