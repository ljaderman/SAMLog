# SAMLog

A lightweight objective-c, thread-safe, (distributed) event log that selectively syncs to the firebase realtime database.

## Synopsis

* "State-And-More Log"... a lightweight, threadsafe, (distributed) event log that selectively syncs to the firebase realtime database.

* It has a notion of "streams" so that different types of data can be logged and other processes can react to logging events (i.e. watch a stream).

* It has a notion of which "device" wrote the entry to the log and locally when the entry was written.

* It can be used to synchronize data between app instances (i.e. play-back distributed log events).

* It can optionally encrypt all data (via RNCryptor, AES-256 CBC) prior to sending it to the cloud.

## Contents

* [Basic Usage](#basic-usage)
* [Installation](#installation)
* [Considerations](#considerations)
* [Design Goals](#design-goals)
* [License](#license)

## Basic Usage

```objc
  // name your streams, for example creating a "dataLog" that logs infrequent data updates which need high-reliability and an "analyticsLog" that records analytics data that we don't care about losing parts of upon app failure
  [[SamLog sharedInstance] addStream:@"dataLog" persistance:SAMLocalPersistanceFile accumulateDelay:2.0 streamType:SAMStreamTypeUser];

  [[SamLog sharedInstance] addStream:@"analyticsLog" persistance:SAMLocalPersistanceMemoryOnly accumulateDelay:10.0 streamType:SAMStreamTypeSystemNonAuth];

  [[SAMLog sharedInstance] startSyncing];
  
  
  ...
  
  // log a simple event...
  [[SAMLog sharedInstance] logEvent:@"analyticsLog" event:@"EVENT_COINS_BY_INAPP_PURCHASE" detail:[NSString stringWithFormat:@"%@", [n stringValue]]];
  
  // log an event with more complex information
  NSDictionary *d = @{
                        @"uid": uid,
                        @"pid": localPlayerId,
                        @"from": fromUser,
                        @"to": playerName,
                        @"amt": [NSString stringWithFormat:@"%ld", amtNum],
                        @"tid": tid
                    };
   [[SAMLog sharedInstance] logEvent:@"dataLog" event:@"EVENT_COINS_TRADED" detailDict:d];
```

You can control where and how data is streamed to firebase.
The "where" is determined by SAMLog_StreamType, as follows:
* SAMStreamTypeUser  - data is written to: /users/{userid}/SAMLog/{stream}/...
* SAMStreamTypeSystemAuth - data is written to: /sys/SAMLog/auth/{stream}/...
* SAMStreamTypeSystemNonAuth - data is written to: /sys/SAMLog/nonauth/{stream}/...

## Installation

### Mobile App - XCode project
Simply copy the .m,.h files into your project.

### Firebase server side
After enbabling the realtime database, edit the realtime database rules allowing SAMLog to write to its expected endpoints, as follows:
```javascript
{
  "rules": {
    "users": {
      "$uid": {
        ".read": "auth != null && ((auth.uid == $uid),
        ".write": "auth != null && ((auth.uid == $uid),
      },
    },
    "sys": {
      "SAMLog": {
        "auth": {
          ".read": "false",
          ".write": "auth != null",
        },
        "noauth": {
          ".read": "false",
          ".write": "true",          
        },
      },
    },
  },
}
```

## Considerations

## Design Goals

SAMLog has several has several design goals, in order of importance:

### Easy to use correctly for most common use cases

The most critical concern is that it be easy to add reasonably-performant reporting of iOS app events that are easy for firebase cloud functions to parse.

### Flexability

SAMLog endeavors to handle the most common app event communication scenarios, such as communicating events without authentication a user and also those after a user has authenticated. 

### Performance

Performance is a goal, but not the most important goal. The code must be easy to use and adapatible to common use-cases. Within that, it is as fast and memory-efficient as possible.

## License

Except where otherwise indicated in the source code, this code is licensed under
the MIT License:

>Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

>The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

>THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NON-INFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE. ```
