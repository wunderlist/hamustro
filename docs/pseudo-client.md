# Pseudo Client for Tracking

These are the functions you have to implement to write a compatible client.

## Initialization

Create a `ClientTracker` object on application open.

```cpp
t = ClientTracker(
  collector_url string, // required, set from config
  shared_secret_key string, // required, set from config
  device_id string, // required, please sha256hex it.
  client_id string, // required
  system_version string, // required
  product_version string, // required
  env, // required
  device_make string,
  device_model string,
  system string,
  system_language string,
  browser string,
  browser_version string,
  product_git_hash string,
  product_language string,
  queue_size int, // set from config, default: 20
  queue_retention int // set from config, default: 1440 (minutes = 24 hours)
)
```

Hamustro client is going to attach the necessary headers when calling the collector but if you need to add extra headers, you can do the following:

```cpp
t.SetHeaders(
  headers map[string]string
)
```

It will generate pre-populated information for new events so it should not be calculated on adding each event.

```cpp
// Generated as md5hex(sha256hex(device_id) + ":" + client_id + ":" + system_version + ":" + product_version + ":" + env)
t.GenerateSession()
```

Loading information from persistent storage.

```cpp
tc.LoadCollections() // not sent events
tc.LoadNumberPerSession() // events per session
tc.LoadLastSyncTime() // timestamp for events sent last time
tc.LoadUser() // tenant_id and user_id
```

Write functions to store the following information also after login:
```cpp
tc.SetUser(tenant_id, user_id)
```

## Track events

To track events you should call the following:

```cpp
t.TrackEvent(
  event string, // required
  params map[string]string
)
```

Please set automatically 
- the `at` uint64 attribute for the events - it must contain an EPOCH UTC timestamp (seconds, ~10 digits),
- `user_id` and `tenant_id` from the persistent storage
- the `timezone` string attribute for actual timzone,
- the `nr` integer attribute - it must contain the serial number of this event within the session all time. So, it starts counting from the first event in the session and it never defaults for that session, not even new application open,
- the `ip` string attribute for actual IPv4 address.
- the `country` string attribute for actual country.

It'll queue up the events within the `ClientTracker`. You should store this in a persistent storage. Please make sure to save the `ClientTracker`'s attributes with the event because you will need to send to the `collector_url` by session.

## Send events to the collector

For sending messages we're using [Protobuf](https://developers.google.com/protocol-buffers/?hl=en) or [JSON](http://www.json.org). This is quicker to handle and smaller than JSON.

This is the message [format](../proto/payload.proto) we're using.

You can send multiple `Payloads`, but as you can see you have to send these by session. Normally you won't have multiple sessions in your code but it can happen with bad connection and updates happening. Please sign them as 'Sent', but don't delete them before getting a response of '200'.

The collector only accepts `POST` with a body of a valid 
- Protobuf bytestream with `application/protobuf` content type,
- unicode JSON string with `application/json` content type. 

Please wait for `200` response code before you delete the already sent payloads. After that update the last sync time. Do not remove the events receiving a different error code.

Please define the following headers for sending:

```
X-Hamustro-Time: EPOCH UTC timestamp
X-Hamustro-Signature: base64(sha256(X-Hamustro-Time + "|" + md5hex(request.body) + "|" + t.shared_secret_key))
Content-Type: application/protobuf or application/json
```

You can check out the proper signature generation in [Python](https://github.com/wunderlist/hamustro/blob/master/utils/message.py#L57-L62).

Send this information to `/api/v1/track`.

## Trigger the sending

It will triggered by the `t.TrackEvent()` function. It is going to send the message to the Collector if
- the unsent payloads number is bigger or equal to the `ClientTracker`'s `queue_size` attribute,
- or the last sync time has happened before current time minus the `ClientTracker`'s `queue_retention`.

Do not send collections without payloads.
