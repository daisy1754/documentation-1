---
title: "Google Cloud Storage Loader"
date: "2020-11-25"
sidebar_position: 500
---


```mdx-code-block
import {versions} from '@site/src/componentVersions';
import CodeBlock from '@theme/CodeBlock';
```

## Overview

Cloud Storage Loader is a [Dataflow](https://cloud.google.com/dataflow/) job which dumps event from an input [PubSub](https://cloud.google.com/pubsub/) subscription into a [Cloud Storage](https://cloud.google.com/storage/) bucket.

## Technical details

Cloud Storage loader is built on top of [Apache Beam](https://beam.apache.org/) and its Scala wrapper [SCIO](https://github.com/spotify/scio).

It groups messages from an input PubSub subscription into configurable windows and write them out to Google Cloud Storage.

It also optionally partitions the output by date so that you can easily see what was outputted when in your cloud storage bucket, for example we could see the following hierarchy:

```bash
gs://bucket/
--2018
----10
------31
--------18
--------19
--------20
```

It can also compress the output data. For now, the supported output compressions are:

- gzip
- bz2
- none

Do note, however, that bz2-compressed data cannot be loaded directly into BigQuery.

For now, it only runs on [GCP's Dataflow](https://cloud.google.com/dataflow/).

### See also:

- [Repository](https://github.com/snowplow-incubator/snowplow-google-cloud-storage-loader/)

## Setup guide

### Configuration

#### Cloud Storage Loader specific options

- `--inputSubscription=String` The Cloud Pub/Sub subscription to read from, formatted as projects/[PROJECT]/subscriptions/[SUB]
- `--outputDirectory=String` The Cloud Storage directory to output files to, ends with /
- `--outputFilenamePrefix=String` Default: output The Cloud Storage prefix to output files to
- `--shardTemplate=String` Default: -W-P-SSSSS-of-NNNNN The shard template which will be part of the filenames
- `--outputFilenameSuffix=String` Default: .txt The suffix of the filenames written out
- `--windowDuration=Int` Default: 5 The window duration in minutes
- `--compression=String` Default: none The compression used (gzip, bz2 or none), bz2 can't be loaded into BigQuery
- `--numShards=int` Default: 1 The maximum number of output shards produced when writing

#### Dataflow options

To run on Dataflow, Beam Enrich will rely on a set of additional configuration options:

- `--runner=DataFlowRunner` which specifies that we want to run on Dataflow
- `--project=[PROJECT]`, the name of the GCP project
- `--streaming=true` to notify Dataflow that we're running a streaming application
- `--workerZone=europe-west2-a`, the zone where the Dataflow nodes (effectively [GCP Compute Engine](https://cloud.google.com/compute/) nodes) will be launched
- `--region=europe-west2`, the region where the Dataflow job will be launched
- `--gcpTempLocation=gs://[BUCKET]/`, the GCS bucket where temporary files necessary to run the job (e.g. JARs) will be stored

The list of all the options can be found at [https://cloud.google.com/dataflow/pipelines/specifying-exec-params#setting-other-cloud-pipeline-options](https://cloud.google.com/dataflow/pipelines/specifying-exec-params#setting-other-cloud-pipeline-options).

### Running

Cloud Storage Loader comes as a ZIP archive, a Docker image or a [Cloud Dataflow template](https://cloud.google.com/dataflow/docs/templates/overview), feel free to choose the one which fits your use case the most.

#### Template

You can run Dataflow templates using a variety of means:

- Using the GCP console
- Using `gcloud`
- Using the REST API

Refer to [the documentation on executing templates](https://cloud.google.com/dataflow/docs/templates/executing-templates) to know more.

Here, we provide an example using `gcloud`:

<CodeBlock language="bash">{
`gcloud dataflow jobs run [JOB-NAME] \\
  --gcs-location gs://sp-hosted-assets/4-storage/snowplow-google-cloud-storage-loader/${versions.gcsLoader}/SnowplowGoogleCloudStorageLoaderTemplate-${versions.gcsLoader} \\
  --parameters \\
    inputSubscription=projects/[PROJECT]/subscriptions/[SUBSCRIPTION],\\
    outputDirectory=gs://[BUCKET]/YYYY/MM/dd/HH/,\\ # partitions by date
    outputFilenamePrefix=output,\\ # optional
    shardTemplate=-W-P-SSSSS-of-NNNNN,\\ # optional
    outputFilenameSuffix=.txt,\\ # optional
    windowDuration=5,\\ # optional, in minutes
    compression=none,\\ # optional, gzip, bz2 or none
    numShards=1 # optional
`}</CodeBlock>

#### ZIP archive

You can find the archive hosted on [Github](https://github.com/snowplow-incubator/snowplow-google-cloud-storage-loader/releases/download/0.3.2/snowplow-google-cloud-storage-loader-0.3.2.zip).

Once unzipped the artifact can be run as follows:

<CodeBlock language="bash">{
`./bin/snowplow-google-cloud-storage-loader \\
  --runner=DataFlowRunner \\
  --project=[PROJECT] \\
  --streaming=true \\
  --workerZone=europe-west2-a \\
  --inputSubscription=projects/[PROJECT]/subscriptions/[SUBSCRIPTION] \\
  --outputDirectory=gs://[BUCKET]/YYYY/MM/dd/HH/ \\ # partitions by date
  --outputFilenamePrefix=output \\ # optional
  --shardTemplate=-W-P-SSSSS-of-NNNNN \\ # optional
  --outputFilenameSuffix=.txt \\ # optional
  --windowDuration=5 \\ # optional, in minutes
  --compression=none \\ # optional, gzip, bz2 or none
  --numShards=1 # optional
`}</CodeBlock>

To display the help message:

```bash
./bin/snowplow-google-cloud-storage-loader --help
```

To display documentation about Cloud Storage Loader-specific options:

```bash
./bin/snowplow-google-cloud-storage-loader --help=com.snowplowanalytics.storage.googlecloudstorage.loader.Options
```

#### Docker image

You can find the image in [Docker Hub](https://hub.docker.com/r/snowplow/snowplow-google-cloud-storage-loader).

A container can be run as follows:

<CodeBlock language="bash">{
`docker run \\
  -v $PWD/config:/snowplow/config \\ # if running outside GCP
  -e GOOGLE_APPLICATION_CREDENTIALS=/snowplow/config/credentials.json \\ # if running outside GCP
  snowplow/snowplow-google-cloud-storage-loader:${versions.gcsLoader} \\
  --runner=DataFlowRunner \\
  --jobName=[JOB-NAME] \\
  --project=[PROJECT] \\
  --streaming=true \\
  --workerZone=[ZONE] \\
  --inputSubscription=projects/[PROJECT]/subscriptions/[SUBSCRIPTION] \\
  --outputDirectory=gs://[BUCKET]/YYYY/MM/dd/HH/ \\ # partitions by date
  --outputFilenamePrefix=output \\ # optional
  --shardTemplate=-W-P-SSSSS-of-NNNNN \\ # optional
  --outputFilenameSuffix=.txt \\ # optional
  --windowDuration=5 \\ # optional, in minutes
  --compression=none \\ # optional, gzip, bz2 or none
  --numShards=1 # optional
`}</CodeBlock>

To display the help message:

<CodeBlock language="bash">{
`docker run snowplow/snowplow-google-cloud-storage-loader:${versions.gcsLoader} \\
  --help
`}</CodeBlock>

To display documentation about Cloud Storage Loader-specific options:

<CodeBlock language="bash">{
`docker run snowplow/snowplow-google-cloud-storage-loader:${versions.gcsLoader} \\
  --help=com.snowplowanalytics.storage.googlecloudstorage.loader.Options
`}</CodeBlock>

#### Additional information

A full list of all the Beam CLI options can be found at: [https://cloud.google.com/dataflow/pipelines/specifying-exec-params#setting-other-cloud-pipeline-options](https://cloud.google.com/dataflow/pipelines/specifying-exec-params#setting-other-cloud-pipeline-options).

### Tests and debugging

#### Testing

The tests for this codebase can be run with `sbt test`.

#### Debugging

You can run the job locally and experiment with its different parts using the [SCIO REPL](https://github.com/spotify/scio/wiki/Scio-REPL) by running `sbt repl/run`.
