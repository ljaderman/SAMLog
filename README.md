# SAMLog - "State-And-More Log"

A lightweight objective-c, thread-safe, (distributed) event log that selectively syncs to the firebase realtime database.

**Solves the problem of quickly communicating app events to firebase cloud-functions and storage with minimal setup.**
Used in production for analytics, a leaderboard, cloud-syncing of calendar app data, and more.

## Synopsis

* A lightweight, threadsafe, (distributed) event log that selectively syncs to the firebase realtime database or a websocket server.

* It has a notion of "streams" so that different types of data can be logged and other processes can react to logging events (i.e. watch a stream).

* It has a notion of which "device" wrote the entry to the log and locally when the entry was written.

* It can be used to synchronize data between app instances (i.e. play-back distributed log events).

* It can optionally encrypt all data (via RNCryptor, AES-256 CBC) prior to sending it to the cloud.

* It can stream to one or more websocket endpoints and attempts to automatically reconnect with randomized exponential backoff.


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
  
  // setup a stream to a local websocket server to log "important" events
  [[SAMLog sharedInstance] addStream:@"localDebug" persistance:SAMLocalPersistanceMemoryOnly accumulateDelay:0.01 streamType:SAMServerStreamTypeWebsocket];

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

You can control where and how data is streamed to.
The "where" is determined by SAMLog_StreamType, as follows:
* SAMStreamTypeUser  - data is written to: /users/{userid}/SAMLog/{stream}/...
* SAMStreamTypeSystemAuth - data is written to: /sys/SAMLog/auth/{stream}/...
* SAMStreamTypeSystemNonAuth - data is written to: /sys/SAMLog/nonauth/{stream}/...

* SAMServerStreamTypeWebsocket - data is sent to a websocket
For websocket streams, bi-directional messaging is supported and SAMLog will pass stream messages on to registered listeners.


## Installation

### Mobile App - XCode project
SAMLog depends upon Firebase and SocketRocket. 

For Firebase, please see the latest installation instructions here: https://firebase.google.com/docs/ios/setup

For SocketRocket, you can find various install options here: https://github.com/facebookincubator/SocketRocket
- **[CocoaPods](https://cocoapods.org)**

 Add the following line to your Podfile:
 ```ruby
 pod 'SocketRocket'
 ```
 Run `pod install`, and you are all set.


Then... simply copy the SAMLog .m,.h files into your project.


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

### Sample websocket server
```python
#!/usr/bin/env python

import asyncio
import json
import websockets
import base64
import sys

USERS = set()

async def register(websocket):
    USERS.add(websocket)
    sys.stdout.write('\a')
    print('> added connection --: ', len(USERS))

async def unregister(websocket):
    USERS.remove(websocket)
    sys.stdout.write('\a')
    print('> removed connection-: ', len(USERS))

async def counter(websocket, path):
    await register(websocket)
    try:
        async for message in websocket:
            data_decoded = base64.b64decode(str(message)).decode('utf-8')
            parsed = json.loads(data_decoded)
            print(json.dumps(parsed, indent=4, sort_keys=True))
    finally:
        await unregister(websocket)

ip='127.0.0.1'
port=8763
print("[[ listening for connections on: " + ip + "::" + str(port) + " ]]")

asyncio.get_event_loop().run_until_complete(
    websockets.serve(counter, ip, port))
asyncio.get_event_loop().run_forever()
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
