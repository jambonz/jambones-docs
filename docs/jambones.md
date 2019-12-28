# Overview 

jambones is a specification for issuing call control commands via JSON messages.  These messages are sent from your application to the jambones platform as instructions for the platform to carry out on specific calls or other resources.  When an incoming call is received by the platform, jmabones makes an HTTP request to the URL endpoint you configured for that dialed number. Your response to that request contains a JSON payload instructing jambones on what to do next.

An outbound call that is initiated by the REST API are controlled in the same way.  When invoking the REST API to launch a call or create a resource, you provide a web callback to receive status on that request.  In your response to that callback, you can provide a JSON payload instructing the platform on what to do next.

## "dial"

The dial command is used to connect an existing call another call.  This is a blocking call; it will reach a completion state when a connected call is ended by a hangup from either side, or the call attempt is rejected, times out trying to connect, or the caller hangs up during the outdial.
```json
  "dial": {
    "action": "http://example.com/outdial",
    "from": "16173331212",
    "timeout": 60,
    "headers": {
      "User-Agent": "my own cpaas",
      "X-Client-ID": "12udih"
    },
    "strategy": "simring",
    "target": [
      {
        "type": "phone",
        "number": "15083084809"
      },
      {
        "type": "sip",
        "uri": "sip:1617333456@sip.trunk1.com",
        "auth": {
          "user": "foo",
          "password": "bar"
        }
      },
      {
        "type": "user",
        "name": "spike"
      }
    ]
  }
```
As the example above illustrates, when you execute the 'dial' command you are making one or more outbound call attempts in an effort to create one new call, which is bridged to the original call.  

When you make multiple attempts, you can use either a 'sequential' strategy (one at a time, in order, until one answers), or a 'simring' strategy (blast outdial them all and accept the first one to answer, hanging up the others).
> Note: when using simring, if a dialed call returns early media (i.e. 183 Session Progress), then at that point all other calls will be canceled and only the call returning the early media will be allowed to complete.

You also have three different types of destinations that you can dial:

* a telephone phone number,
* a sip endpoint, identified by a sip uri, or
* a webrtc or sip client that has registered directly with your application.

You can use the following attributes in the `dial` command:

| option        | description | required  |
| ------------- |-------------| -----|
| from | caller id to use on the outbound call; when calling to a PSTN number it must be one of your jambones numbers | yes, if calling to PSTN |
| timeout | ring no answer timeout | no (default: 60 secs) |
| action | webhook to invoke when call ends | no (default: current document url) |
| headers | an object containing arbitrary sip headers to apply to the outbound call attempt(s) | no |
| strategy | "simring" or "hunt" | no (default: "hunt" if multiple targets provided) |
| target | array of targeted [endpoints](#target-types) | yes |
| transcribe | a nested [transcribe](#transcribe) action, which will cause the call to be transcribed | no |
| listen | a nested [listen](#listen) action, which will cause audio from the call to be streamed to a remote server over a websocket connection | no |

##### target types

*PSTN number*

| option        | description | required  |
| ------------- |-------------| -----|
| type | phone | yes |
| number | E.164 number | yes |

*sip endpoint*

| option        | description | required  |
| ------------- |-------------| -----|
| type | sip | yes |
| uri | sip uri | yes |
| auth | authentication credentials | no |
| auth.username | sip username | no |
| auth.password | sip password | no |

Using this approach, it is possible to send calls out a sip trunk.  If the sip trunking provider enforces username/password authentication, then supply the credentials in the `auth` property.

**a registered webrtc or sip user**

| option        | description | required  |
| ------------- |-------------| -----|
| type | user | yes |
| name | registered username | yes |

Note that you do not need to specify the fully-qualified address of record (e.g. `spike@myapp.myco.drachtio.org`); the domain is automatically applied to the name provided when searching active registrations.


## "gather"

The gather command is used to collect dtmf or speech input.  gather is a blocking command.

```json
"gather": {
  "input": "dtmf",
  "finishOnKey": "#",
  "numDigits": 5,
  "timeout": 8,
  "action": "http://example.com/collect"
}
```

You can use the following options in the `gather` command:

| option        | description | required  |
| ------------- |-------------| -----|
| action | web callback to invoke with collected digits or speech | no (defaults to current document url) |
| actionOnEmptyResult | whether to invoke the callback when no input is detected | no (default: false) |
| finishOnKey | dmtf key that signals the end of input | no |
| hints | array of words or phrases to help speech detection | no |
| input | type of input: 'dtmf', 'speech', or ['dtmf', 'speech'] | no (default: dtmf)|
| language | language code to use for speech detection | no (default: en-US) |
| numDigits | number of dtmf digits expected to gather | no |
| partialResultCallback | url to send interim transcription results to (may not return content) | no |
| speechTimeout | seconds to wait for a final speech transcription | no |

## "hangup"

The hangup command terminates the call and ends the application.
```json
"hangup": {
  "headers": {
    "X-Reason" : "maximum call length detected"
  }
}
```

You can use the following options in the `playback` action:

| option        | description | required  |
| ------------- |-------------| -----|
| headers | an object containing SIP headers to include in the BYE request | no |


## "listen"

The listen action is used to stream audio in real-time to a server over a websocket connection.  The audio format will be linear 16-bit PCM encoding with a configurable sample rate. The websocket subprotocol name will be `audio.jambones.org`.  The listen action is a non-blocking action.

(*TBD: link for more detail on the protocol, json metadata etc*)

A listen command can also be nested in a [dial](#dial) command, which allows the audio for the call to be streamed in real-time to a web service.


```json
"listen": {
  "action": "http://example.com/listen",
  "wsUrl": "wss://myrecorder.example.com:4433",
  "mixType" : "stereo",
  "sampleRate": 8000,
  "metadata": {
    "clientId": "12udih"
  }
}
```

You can use the following options in the `listen` action:

| option        | description | required  |
| ------------- |-------------| -----|
| action | web callback to invoke with status of websocket connection | no |
| wsUrl | url of remote server to connect to | yes |
| mixType | "mono" (send single channel), "stereo" (send dual channel of both calls in a bridge), or "mixed" (send audio from both calls in a bridge in a single mixed audio stream) | no (default: mono) |
| sampleRate | sample rate of audio to send (allowable values: 8000, 16000, 24000, 48000, or 64000 | no (default: 8000) |
| metadata | arbitrary JSON payload to send to remote server when websocket connection is established | no |

## "play"

The play command is used to stream recorded audio to a call.  This is a non-blocking call.
```json
"play": {
  "url": "https://example.com/example.mp3",
  "bargein": true,
  "loop": 2
}
```

You can use the following options in the `playback` action:

| option        | description | required  |
| ------------- |-------------| -----|
| url | a single url or array of urls (will play in sequence) to a wav or mp3 file | yes |
| bargein | if true, terminate playback when a dtmf entry is detected | no (default: false) |
| loop | number of times to play the url(s), or "forever" to loop indefinitely | no (default: 1) |

## "redirect"

The redirect action is used to transfer control to another JSON document taht is retrieved from the specified url.  All actions after `redirect` are unreachable and ignored.

```json
"redirect": {
  "url": "https://example.com/?foo=bar"
}
```

You can use the following options in the `redirect` action:

| option        | description | required  |
| ------------- |-------------| -----|
| url | url to retrieve document from | yes |

## "sip:decline"

The sip:decline action is used to reject an incoming call with a specific status.  This action must be the first and only action returned in the web callback for an incoming call, and the application must be configured to park incoming calls rather than automatically answer them.  The sip:decline action is a non-blocking action and the session ends immediately after the action is executed.

```json
"sip:decline": {
  "status": 480,
  "reason": "Gone Fishing"
}
```

You can use the following options in the `sip:decline` action:

| option        | description | required  |
| ------------- |-------------| -----|
| status | a valid SIP status code in the range 4XX - 6XX | yes |
| reason | a brief description | no (default: the well-known SIP reasons associated with the specified status code |

## "sip:notify"

The sip:notify action is used to a SIP NOTIFY request to one or more registered users.  The sip:notify action is a non-blocking action.

```json
"sip:notify": {
  "user": ["spike"],
  "contentType": "application/simple-message-summary",
  "content": [
    "Messages-Waiting: yes",
    "Message-Account: sip:spike@sip.example.com",
    "Voice-Message: 2/8 (0/2)"
  ],
  "headers": {
    "Subscription-State": "active"
  }
}
```

You can use the following options in the `sip:notify` action:

| option        | description | required  |
| ------------- |-------------| -----|
| user | user, or array of users, to send NOTIFY requests to | yes |
| contentType | value of SIP Content-Type header in NOTIFY | yes |
| content | body of NOTIFY request | yes |
| headers | object specifying arbitrary headers to apply to SIP NOTIFY request | no

## "sip:redirect"

The sip:redirect command is used to redirect an incoming call to another sip server or network.  This must be the first and only command returned in the web callback for an incoming call.  A SIP 302 Moved response will be sent back to the caller.  

The sip:redirect command is a non-blocking and the session ends immediately after the command is executed.

```json
"sip:redirect": {
  "sipUri": "sip:123@gateway.example.com"
}
```

You can use the following options in the `sip:decline` command:

| option        | description | required  |
| ------------- |-------------| -----|
| sipUri | a sip uri that will appear in the Contact header of the 302 response to the caller | yes |

## "talk"

The talk command is used to send synthesized speech to the remote party. The text provided may be either plain text or may use SSML tags.  This is a non-blocking action.

A talk command can also be nested in a [dial](#dial) command, which allows the complete audio stream for the call to be streamed in real-time to a web service over websockets.

```json
"talk": {
  "text": "hi there!",
  "vendor" : "google",
  "voice": "en-AU-Wavenet-B",
  "bargein": false
}
```

You can use the following options in the `talk` action:

| option        | description | required  |
| ------------- |-------------| -----|
| text | text to speak; may contain SSML tags | yes |
| vendor | speech vendor to use| no (default: google) |
| voice | name of voice to use; note this is vendor-dependent  | yes |

*TBD: list of speech vendors supported (currently only google)*

[list of google voices](#google-tts-voices)

## "transcribe"

The transcribe command is used to send real time transcriptions of speech (either caller, called party, or both) to your web callback.

This is a non-blocking command.  When your web callback is invoked for a final transcription, it can optionally return a JSON document with further instructions; if provided, this will cause the current document to be flushed and execution to continue with the new document provided.

A transcribe command can also be nested in a [dial](#dial) command, which allows a long-running transcription to be triggered for an outbound call.

```json
"transcribe": {
  "action": "http://example.com/transcribe",
  "language" : "en-US",
  "target" : "both",
  "interim": true,
  "vendor": "google"
}
```

You can use the following options in the `transcribe` command:

| option        | description | required  |
| ------------- |-------------| -----|
| action | web callback to invoke with transcriptions | yes |
| language | language to use for speech transcription | yes |
| target | call uuid to perform transcription on, or "both" to transcribe both calls in a bridge | no (default: original call) |
| interim | whether to send interim transcriptions | no (default: false) |
| vendor | speech vendor to use | no (default: google) |
