---


---

<h1 id="arriving-home----play-to-alexa-multi-room-audio-setup">Arriving Home -  Play to Alexa Multi Room Audio Setup</h1>
<p><img src="https://i.imgur.com/cuPjrpG.png" alt="Arriving Home - Whole Home Alexa Audio"></p>
<p>This is building off of <a href="https://github.com/davearneson/alexawholehome">this previous project of mine</a> to create a system of automations to automatically play a variety of music for me when I get home.</p>
<h2 id="alexa-media-player-setup">Alexa Media Player Setup</h2>
<p>This project uses the <a href="https://github.com/custom-components/alexa_media_player">Alexa Media Player</a> custom component to interact with my Alexa Devices. I won’t go into detail on setting this up, because I haven’t deviated any from the standard setup for this custom component.</p>
<p>As I mentioned in the previous project, <a href="https://github.com/custom-components/alexa_media_player/wiki#play-in-alexa-groups">there is currently a limitation to Alexa Media Player</a> which means you can start and stop music to individual echo devices, but you can’t send music directly to an Alexa Multi-Room Music Group. The workaround that I’ve found to be most consistent - which I use in this project - is sending a custom simulated voice command to Alexa through the component, like “Play Jazz on the Downstairs Music Group.”</p>
<h2 id="home-assistant-helpers">Home Assistant Helpers</h2>
<p>I used HA’s helpers to set up an input_text entity in the UI by going to Configuration/Helpers and clicking the add button, then “Text.” I then added a name, in my case “Arriving Home Music”. This entity will be used to type text for whatever music or audio I want sent to the Alexa Music Groups. I changed the icon to “mdi:music”</p>
<p><img src="https://i.imgur.com/hDv1B0q.png" alt="Helper Setup for Arriving Home - Whole Home Alexa Audio"></p>
<h1 id="actionable-notification">Actionable Notification</h1>
<p>I have an iPhone, so this part of the uses actionable notifications for iOS. It’s fairly well documented here: <a href="https://companion.home-assistant.io/docs/notifications/actionable-notifications/">https://companion.home-assistant.io/docs/notifications/actionable-notifications/</a></p>
<p><img src="https://i.imgur.com/QmAIbKf.jpg" alt="Arriving Home Music Actionable Notification"></p>
<h2 id="configuration.yaml">configuration.yaml</h2>
<p>I added one section to configuration.yaml for Actionable Notifications for this project:</p>
<pre><code># configuration.yaml

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
</code></pre>
<p>The “textInput” behavior allows me to respond inside the actionable notification with text that will be sent to Home Assistant:</p>
<p><img src="https://i.imgur.com/ZlgSEsK.jpg" alt="Arriving Home Music Actionable Notification"></p>
<h1 id="automations">Automations</h1>
<p>I created three automations for this project, and I’ll do my best to explain them and the logic behind them:</p>
<h2 id="what-music-do-you-want-notification">What Music Do You Want? Notification</h2>
<p>The first few automation sends the actionable notification at prescribed time, about 15 minutes before I leave work each weekday.</p>
<pre><code># automations.yaml

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
</code></pre>
<p><img src="https://i.imgur.com/ZMAB6TL.jpg" alt="Arriving Home Music Actionable Notification"></p>
<h2 id="automation-components">Automation Components</h2>
<p>This automation to process the return from the Actionable Notification is where it gets fun. I set the triggers for this one up in the UI to make things easier - there are triggers for each of the actionable notification returns I set up earlier (JAZZ_MUSIC, CLASSICAL_MUSIC,  RANDOM_MUSIC, and INPUT_MUSIC). Then I set up the action to fire a service, using as an if statement as the data value (I had to set this up in YAML, because it uses “data_template”). This is using the input_text.set_value service to change the helper I created earlier to whatever my response from the actionable notification was.</p>
<pre><code># automations.yaml

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
	  value: &gt;-
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
</code></pre>
<h2 id="play-music-automation">Play Music Automation</h2>
<p>Finally, this last automation is what sets off the actual music when I get home from work. It’s simply using HA zones and my iOS device to see when I enter the “home” zone. I have it time-conditioned, and conditioned for when my spouse is not already home, because they would probably not appreciate being blasted with my music if they’re home watching TV or just generally minding their own business.</p>
<pre><code># automations.yaml

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
</code></pre>
<h1 id="scripts">Scripts</h1>
<p>I also created one script for use in this project. This is an (overkill) fail-safe to make sure all music and audio stops if I want it to stop. I made this as a script so I could control it through automations and manual input buttons in the UI.</p>
<pre><code>##############################################################################        
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
</code></pre>
<p><img src="https://i.imgur.com/wuJB5lk.png" alt="Stop Music as a button in Lovelace"><br>
This one is just sending “stop music” commands to all possible groups.</p>
<h1 id="final-thoughts">Final Thoughts</h1>
<p>I went back and forth on whether I wanted to create another automation to reset the music choice each day, or if I wanted to be able to leave it on a selection for as long as I feel like it. For now, I’m sticking with letting the previous day’s choice hold over until I decide to change it. I’ve had this set up for about a week so far, and overall, I’m really happy with the setup.</p>

