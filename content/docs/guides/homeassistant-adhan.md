---
title: "Home Assistant Adhan Automation"
date: 2025-06-02
weight: 20
draft: false
---

# Home Assistant Adhan Automation

## Overview

{{< hint info >}}
This guide covers setting up an automated Adhan system in <a href="https://home-assistant.io">Home Assistant</a>
{{< /hint >}}


## Prerequisites

- Home Assistant ofcourse!
- Islamic prayer times integration
- Wireless media player
- Audio files in Home Assistant media folder

## Setup Steps

### 1. Install Islamic Prayer Times Integration

Configure the Islamic Prayer Times Integration to use a calculation method that is accurate.

{{< hint warning >}}
Prayer times from Islamic Prayer Times Integration method are returned in UTC and require timezone offset adjustment.
{{< /hint >}}

### 2. Create Input Number Helper

* Create a helper as **input_number**
* Name the helper **adhan_offset_hours**
* Give it a value that fits your reigon offset time from **UTC**

### 3. Add the date time integration

Enable it in the **integration** page the automation requires it to function

### 4. Upload Audio Files
Upload the audio files in the **Media** section in the GUI

We have two seperate audio files:
* adhan.mp3
* fajr.mp3


### 5. Create the Automation

```yaml
alias: Play Adhan for All Prayers (offset helper)
description: >
  Plays the Adhan on a wireless speaker for all five prayers, using different
  volume/file for Fajr and offset controlled by an input_number.
triggers:
  - id: fajr
    value_template: |
      {{ states('sensor.date_time_iso')
         == (as_datetime(states('sensor.islamic_prayer_times_fajr_prayer')) 
             + timedelta(hours= states('input_number.adhan_offset_hours')|float ))
            .strftime('%Y-%m-%dT%H:%M:%S') }}
    trigger: template
  - id: dhuhr
    value_template: |
      {{ states('sensor.date_time_iso')
         == (as_datetime(states('sensor.islamic_prayer_times_dhuhr_prayer')) 
             + timedelta(hours= states('input_number.adhan_offset_hours')|float ))
            .strftime('%Y-%m-%dT%H:%M:%S') }}
    trigger: template
  - id: asr
    value_template: |
      {{ states('sensor.date_time_iso')
         == (as_datetime(states('sensor.islamic_prayer_times_asr_prayer')) 
             + timedelta(hours= states('input_number.adhan_offset_hours')|float ))
            .strftime('%Y-%m-%dT%H:%M:%S') }}
    trigger: template
  - id: maghrib
    value_template: |
      {{ states('sensor.date_time_iso')
         == (as_datetime(states('sensor.islamic_prayer_times_maghrib_prayer')) 
             + timedelta(hours= states('input_number.adhan_offset_hours')|float ))
            .strftime('%Y-%m-%dT%H:%M:%S') }}
    trigger: template
  - id: isha
    value_template: |
      {{ states('sensor.date_time_iso')
         == (as_datetime(states('sensor.islamic_prayer_times_isha_prayer')) 
             + timedelta(hours= states('input_number.adhan_offset_hours')|float ))
            .strftime('%Y-%m-%dT%H:%M:%S') }}
    trigger: template
actions:
  - data:
      entity_id: media_player.your_speaker_entity
      volume_level: |
        {% if trigger.id == 'fajr' %}
          0.9
        {% else %}
          0.8
        {% endif %}
    action: media_player.volume_set
  - data:
      entity_id: media_player.your_speaker_entity
      media_content_id: |
        {% if trigger.id == 'fajr' %}
          media-source://media_source/local/fajr.mp3
        {% else %}
          media-source://media_source/local/adhan.mp3
        {% endif %}
      media_content_type: audio/mpeg
    action: media_player.play_media
mode: single
```

## Notes:
*  Use the correct media entity in the automation
*  Verify the timezone offset is correct
*  Verify the calculation method that is used is accurate
