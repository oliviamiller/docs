---
title: "Sync captured data to the cloud with cloud sync"
linkTitle: "Cloud Sync"
description: "Configure cloud sync to automatically capture data in the Viam app."
weight: 15
type: "docs"
tags: ["data management", "cloud", "sync"]
icon: true
images: ["/services/icons/data-cloud-sync.svg"]
aliases:
  - "/services/data/cloud-sync/"
  - "/data/cloud-sync/"
no_service: true
# SME: Alexa Greenberg
---

The data management service securely syncs the specified data to the cloud at the user-defined frequency.
Viam does not impose a minimum or maximum on the frequency of data syncing.
However, in practice, your hardware or network speed may impose limits on the frequency of data syncing.

If the internet becomes unavailable or the machine needs to restart during the sync process, the service will try to resume sync indefinitely.
When the connection is restored, the service resumes the syncing process where it left off without duplicating data.
For more detailed information, see [Considerations](#considerations).

Once the service syncs a file to Viam's cloud, the service deletes the file locally from the machine's configured capture location.

As before, consider the example of a tomato picking robot.
When you initially set the robot up you may want to sync captured data to the cloud every five minutes.
If you change your mind and want your machine to sync less frequently, you can change the sync frequency, for example, to once a day.

## Configuration

Before you can configure [cloud sync](/services/data/cloud-sync/), you must [add the data management service](/services/data/capture/#add-the-data-management-service).

To enable cloud sync, navigate to your machine's **CONFIGURE** tab and enable **Syncing** for your [data management service](../).
Click the **Save** button in the top right corner of the page.

Now the data that you capture will sync automatically with the Viam app in the cloud.

![data capture configuration](/tutorials/data-management/data-management-conf.png)

By default, the data management service syncs data to Viam's cloud every 0.1 minutes, that is every 6 seconds.
To change the sync interval, specify an interval in minutes in the interval field.

{{%expand "Click to view the JSON configuration for the data management service" %}}

```json {class="line-numbers linkable-line-numbers"}
{
  "components": [],
  "services": [
    {
      "name": "data_manager",
      "type": "data_manager",
      "attributes": {
        "sync_interval_mins": 0.1,
        "capture_dir": ""
      }
    }
  ]
}
```

{{< alert title="Info" color="info" >}}
If `capture_dir` is unspecified, `viam-server` will use the default directory at <file>~/.viam/capture</file>.
{{< /alert >}}

{{% /expand%}}

### Trigger sync conditionally

You can use a {{< glossary_tooltip term_id="module" text="module" >}} to sync data only when a certain logic condition is met, instead of at a regular time interval.
For example, if you rely on mobile data but have intermittent WiFi connection in certain locations or at certain times of the day, you may want to trigger sync to only occur when these conditions are met.
To set up triggers for syncing see [Trigger Sync](/services/data/trigger-sync/).

### Configure sync threads

When cloud sync is enabled, the data management service will use up to `1000` concurrent CPU threads by default to sync data to the Viam cloud, depending on how much data is being synced.
You can adjust the permitted thread count with the `maximum_num_sync_threads` attribute.

The default value of `1000` concurrent threads is sufficient for most use cases, but if you are using limited hardware, are operating under heavy CPU load, or are syncing a large amount of data at once, consider lowering this value as needed.

{{%expand "Click to view the JSON configuration with 250 threads configured" %}}

```json {class="line-numbers linkable-line-numbers"}
{
  "components": [],
  "services": [
    {
      "name": "data_manager",
      "type": "data_manager",
      "attributes": {
        "sync_interval_mins": 0.1,
        "capture_dir": "",
        "maximum_num_sync_threads": 250
      }
    }
  ]
}
```

{{% /expand%}}

### Pause sync

You can pause Cloud Sync at any time by navigating to your machine's **CONFIGURE** tab and disabling **Syncing** for your [data management service](../).
If you have captured data that you do not want to sync, delete the data on the machine before resuming Cloud Sync.
To delete the data locally, `ssh` into your machine and delete the data in the directory where you capture data.

## Sync files from another directory

You may have additional files you want to sync to the cloud from your machine.
For example, there may be components on your machine which are not controlled by Viam that are collecting data locally on your machine.
Or there may be a set of logs indicating the status of the machine at different points in time.
To include these types of files in cloud sync, click `ADD PATHWAY` in the data management service panel and specify the directory where your files are located on your machine.
Once you save the configuration, the data management service begins syncing the files in the specified folder at the interval configured for the service.
To avoid syncing files that are still being written to, the data management service only syncs files that haven't been modified in the previous 10 seconds.

{{< alert title="Caution" color="caution" >}}
If a machine does not write to a file for 10 seconds, the data management service syncs the file and deletes it from the machine.
{{< /alert >}}

{{< alert title="Info" color="tip" >}}
Currently, if the internet becomes unavailable and the sync is interrupted mid-file, the service resumes sync from the beginning of the file.
This is only applicable for files in a directory added as an additional sync path.
{{< /alert >}}

In the example pictured here, the data management service syncs the configured component data from `/.viam/capture` as well as all files in `/logs` every .1 minutes.

![service config example](/services/data/data-service-config.png)

{{%expand "Click to view the JSON configuration for this example" %}}

```json {class="line-numbers linkable-line-numbers"}
{
  "components": [],
  "services": [
    {
      "name": "data_manager",
      "type": "data_manager",
      "attributes": {
        "sync_interval_mins": 0.1,
        "capture_dir": "",
        "sync_disabled": false,
        "additional_sync_paths": ["/logs"]
      }
    }
  ]
}
```

{{% /expand%}}

## Considerations

- **Security**: The data management service uses {{< glossary_tooltip term_id="grpc" text="gRPC" >}} calls to send and receive data, so your data is encrypted while in flight.
  When data is stored in the cloud, it is encrypted at rest by the cloud storage provider.

- **Data Integrity**: Viam's data management service is designed to safeguard against data loss, data duplication and otherwise compromised data.

  If the internet becomes unavailable or the machine needs to restart during the sync process, the sync is interrupted.
  If the sync process is interrupted, the service will retry uploading the data at exponentially increasing intervals until the interval in between tries is at one hour at which point the service retries the sync every hour.
  When the connection is restored and sync resumes, the service continues sync where it left off without duplicating data.

  For example, if the service has uploaded 33% of the data and then the internet connection is severed, sync is interrupted.
  Once the service retries and successfully connects, data synchronization resumes at 33%.

- **Storage** Data that is successfully synced to the cloud is automatically deleted from local storage.

  When a machine loses its internet connection, it cannot resume cloud sync until it can reach the Viam cloud again.

  To ensure that the machine can store all data captured while it has no connection, you need to provide enough local data storage.

  {{< alert title="Warning" color="warning" >}}

  If your machine's disk fills up beyond a certain threshold, the data management service will delete captured data to free up additional space and maintain a working machine.
  See [automatic data deletion](/services/data/capture/#automatic-data-deletion) for configuration options.

  {{< /alert >}}

## Next steps

To view your captured data in the cloud, see [View Data](/services/data/view/).

To export your captured data from the cloud, see [Export Data](/services/data/export/).

If you have synced data, such as [sensor](/components/sensor/) readings, you can [query that data with SQL or MQL](/use-cases/sensor-data-query/) from the Viam app or a MQL-compatible client.
If you have synced images, you can use those images to [train machine learning models](/services/ml/train-model/) within the Viam app.

For a comprehensive tutorial on using data capture and synchronization together with the ML model service, see [Capture Data and Train a Model](/tutorials/services/data-mlmodel-tutorial/).
