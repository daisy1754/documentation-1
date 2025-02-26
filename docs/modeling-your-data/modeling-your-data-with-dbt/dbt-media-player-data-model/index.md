---
title: "The Media Player data model"
date: "2022-10-05"
sidebar_position: 103
---

# Snowplow Media Player Package

**The package source code can be found in the [snowplow/dbt-snowplow-media-player repo](https://github.com/snowplow/dbt-snowplow-media-player), and the docs for the [model design here](https://snowplow.github.io/dbt-snowplow-media-player/#!/overview/snowplow_media_player).**

The package is built as an extension of the [dbt-snowplow-web package](/docs/modeling-your-data/modeling-your-data-with-dbt/dbt-web-data-model/index.md) that transforms raw media player event data into derived tables for easier querying generated by the Snowplow [JavaScript tracker](/docs/collecting-data/collecting-from-own-applications/javascript-trackers/index.md) in combination with media tracking specific plugins such as the [Media Tracking plugin](/docs/collecting-data/collecting-from-own-applications/javascript-trackers/javascript-tracker/javascript-tracker-v3/plugins/media-tracking/index.md) or the [YouTube Tracking plugin](/docs/collecting-data/collecting-from-own-applications/javascript-trackers/javascript-tracker/javascript-tracker-v3/plugins/youtube-tracking/index.md).

In order to keep the documentations separate and less verbose, this guide will assume the reader is already familiar with configuring and running the web model and will only explain how to operate the media-player package in conjunction with the web model as the media model was designed to be run together with it. Please refer to the [snowplow-web dbt docs](/docs/modeling-your-data/modeling-your-data-with-dbt/index.md) for a full breakdown of the package and how to set it up.

:::caution
Please note that the media player package is not compatible with the [Flutter tracker](/docs/collecting-data/collecting-from-own-applications/flutter-tracker/index.md) as it is reliant on the Snowplow [JavaScript tracker](/docs/collecting-data/collecting-from-own-applications/javascript-trackers/index.md).

:::

## Overview

This package consists of a series of dbt models with the goal to produce the following main aggregated models from the raw media player events and relevant contexts:

  - `snowplow_media_player_base`: This derived table summarizes the key media player events and metrics of each media element on a media_id and pageview level which is considered as a base aggregation level for media interactions.

  - `snowplow_media_player_plays_by_pageview`: This view removes impressions from the '_base' table to summarize media plays on a page_view by media_id level.

  - `snowplow_media_player_media_stats`: This derived table aggregates the '_base' table to individual media_id level, calculating the main KPIs and overall video/audio metrics.

The package is built on top of the [dbt-snowplow-web package](/docs/modeling-your-data/modeling-your-data-with-dbt/dbt-web-data-model/index.md) taking that as a basis to carry out the incremental update. It is designed to be run together with the web model in a similar manner to how a custom module would run.

The  `_interactions_this_run` table takes the `snowplow_web_base_events_this_run` table generated by the web package as an input then adds the various contexts to enrich the base table with the additional media related fields. It could be used for custom models for more in-depth event level derived tables and further analysis.

The  `_base_this_run` table then aggregates the `_interactions_this_run` table to media_id and pageview level and serves as a basis for the incrementalized derived table `_media_base`.

The main `_media_stats` derived table will also be updated incrementally based on the `_media_base` derived table, however not through the `snowplow_incremental` materialization, but using the native dbt incremental materialization on a pageview basis after a set time window passed. This is to prevent complex and expensive queries due to metrics which need to take the whole page_view events into calculation. This way the metrics will only be calculated once per pageview / media, after no new events are expected.

The additional `_pivot_base` table is there to calculate the percent_progress boundaries and weights that are used to calculate the total play_time and other related media fields.


# Custom models

There are two custom models included in the package which could potentially be used in downstream models:

1. the `snowplow_media_player_session_stats` table, which aggregates the snowplow_media_base table on a session level

2. the `snowplow_media_player_user_stats` table, which aggregates the snowplow_media_player_session_stats to user level

By default these are disabled, but you can enable them in the project's `profiles.yml`, if needed.

```yml
# dbt_project.yml
...
  models:
    snowplow_media_player:
      custom:
        enabled: true
```
Just like in case of the web model, users are encouraged to use the Media Player model and its incremental logic to design their own custom models / modules. The `snowplow_media_player_interactions_this_run` table is designed with this in mind, where a couple of potentially useful fields are generated that the Media Player model does not use downstream but they nonetheless have the potential to be incorporated into users custom models.

One such example is the `player_current_time`, which is the playback position of a specific media in seconds whenever a media player event is fired, from which more precise time-based calculations could be made.

e.g. subsequent events' `player_current_time` could be used to deduct the `player_end_time` of an event. Subtracting these two would result in calculated play_times instead of taking the percent_progress fields as a base like the Media Player model.

```sql
with interaction_ends as (

   select
     event_id,
     event_type,
     start_tstamp,
     lead(start_tstamp, 1) over(partition by play_id order by start_tstamp) as end_tstamp,
     player_current_time,
     lead(player_current_time, 1) over(partition by play_id order by start_tstamp) as player_end_time
from scratch.snowplow_media_player_interactions_this_run

)

select
	event_id,
	player_end_time - player_current_time as play_time_sec_calculated

from interaction_ends

where event_type = 'play'
```
