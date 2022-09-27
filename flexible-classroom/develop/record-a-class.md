---
title: "Record a class"
sidebar_position: 6
type: docs
description: >
    Best practice to implement the recording feature in Flexible Classroom
---

This page introduces some best practices that you need to keep in mind when you implement the recording feature in Flexible Classroom. These best practices can help to ensure the reliability of the recording and improve the quality of the recorded files.

## The process of implementing the recording feature

The following figure shows the basic process of implementing the recording feature in Flexible Classroom. The steps marked in purple in the figure are the operations you need to perform.

![](https://web-cdn.agora.io/docs-files/1638259107675)

## Start the recording

Whether you initiate the recording on the client or server, call [Set the recording state](../reference/classroom-api#set-the-recording-state). When calling this method, pay attention to the following parameters:

- `mode`: Set this parameter as `web` to enable [web page recording](../../cloud-recording/develop/webpage-mode).
- `rootUrl`: The root address of the web page to be recorded. The Flexible Classroom cloud service automatically gets the full address of the web page to be recorded by appending `roomUuid`, `roomType`, and other parameters after the root address. You need to extract this information from the URL and pass it in when calling the [launch](../reference/classroom-sdk#launch) method.
- `retryTimeout`: The amount of time (seconds) that the Flexible Classroom cloud service waits between attempts to begin recording. The Flexible Classroom cloud service retries a maximum of two times.

Sample code:

```json
{
    "mode": "web",
    "webRecordConfig": {
        "rootUrl":"https://xxx.yyy.zzz",
    },
    ...
    "retryTimeout": 60
}
```

After setting `retryTimeout`, when calling the [launch](../reference/classroom-sdk#launch) method, you need to set the `listener` parameter to listen for the `1` event, which represents that the page has been loaded. When this event is triggered, you need to call the following method to inform the Flexible Classroom cloud service. If the Flexible Classroom cloud service does not receive this notification within `retryTimeout`, it retries the recording.

- Prototype

  - Method: PUT

  - Path: https://api.agora.io/edu/apps/{appId}/v2/rooms/{roomUUid}/records/ready

  - A successful method call returns the following response:

    ```json
    status: 200,
    body:{
        "msg": "Success",
        "code": 0,
        "ts": 1610167740309
    }
    ```

The following sample code shows this logic:

```typescript
AgoraEduSDK.launch(document.querySelector(`#${this.elem.id}`), {
    ...
    listener: (evt) => {
        if ( evt === 1 ) {
        // Send a web request to notify the server side that the recording page was fully loaded.
        }
    }
})
```

If you use a version earlier than v1.1.5, you also need to set `params` to monitor the whiteboard loading state. The following sample code shows this logic:

```typescript
AgoraEduSDK.launch(document.querySelector(`#${this.elem.id}`), {
    ...
    listener: (evt, params) => {
        if ( evt === 1 && params.type === "whiteboard") {
        // Send a web request to notify the server side that the recording page was fully loaded.
        }
    }
})
```

## Get the recording state

After starting the recording, the Flexible Classroom cloud service generates an event to indicate the [recording state change](../reference/classroom-api#the-recording-state-changes). You can get the recording state by calling [Query a specified event](../reference/classroom-api#query-a-specified-event) or [Get classroom events](../reference/classroom-api#get-classroom-events). Pay attention to the `reason` field in the recording state change event:

- `1`: Start the recording normally.
- `2`: Stop the recording normally.
- `3`: Start the recording after retry.
- `4`: Time out. Wait for retry.
- `5`: Exit the recording when the number of retries reaches the upper limit.

You can further implement your own logic based on the recording state change.

## Remove the white screen at the beginning of the recorded file

It takes a while for the recording server to load the web page, but the file slicing begins before the loading finishes. As a result, there may be a period of white screen at the beginning of the recorded file. To remove the white screen, do the following:

1. Before the class starts, call [Set the recording state](../reference/classroom-api#set-the-recording-state), and set `onhold` as `true`. The Flexible Classroom cloud service pauses the recording immediately after the recording task is initiated. The recording server opens and renders the web page, but does not generate a slice file. The following sample code shows this logic:

   ```json
   {
       "mode": "web",
       "webRecordConfig": {
           "url": "https://xxx.yyy.zzz",
           "onhold": true
       }
   }
   ```

2. At least 60 seconds later, call [Update the recording configurations](../reference/classroom-api#update-the-recording-configurations) and set the `onhold` parameter as `false` to start the recording and file slicing. The following sample code shows this logic:

   ```json
   {
       "webRecordConfig": {
           "onhold": false
       }
   }
   ```

## Improve the video clarity when the recorded content is a shared screen

In scenarios where the recorded content is a shared screen or whiteboard, if you have high requirements for video clarity, you can set the following parameters when calling [Set the recording state](../reference/classroom-api#set-the-recording-state):

- Set `videoWidth` as 1920.
- Set `videoHeight` as 1080.
- Set `videoBitrate` as 2000.

Request body:

```json
{
    "mode": "web",
    "webRecordConfig": {
        "url": "https://xxx.yyy.zzz",
        "videoWidth": 1920,
        "videoHeight": 1080,
        "videoBitrate": 2000
    }
}
```