
# SendBird SyncManager for Android

SendBird SyncManager is chat data sync management library for SendBird. SyncManager offers an event-based data management framework so that each view would see a single spot by subscribing data event. And it stores the data into SQLite which implements local caching for faster loading.

## Install using Gradle

```
repositories {
    maven { url "https://raw.githubusercontent.com/smilefam/sendbird-syncmanager-android/master/" }
}
dependencies {
    compile 'com.sendbird.sdk:sendbird-syncmanager:1.0.0'
}
```

## Sample

We provide sample project to understand `SyncManager` further. Check out [SyncManager sample](https://github.com/smilefam/SendBird-Android/tree/master/syncmanager).

## How It Works

### Initialization

```java
SendBirdSyncManager.setUp(getApplicationContext(), userId, new CompletionHandler() {
    @Override
    public void onCompleted(SendBirdException e) {
        if (e != null) {
            // Error handling
           return;
        }
    }
});
```

### Collection

Collection is a component to manage data related to a single view. `ChannelCollection` and `MessageCollection` are attached to channel list view and message list view (or chat view) accordingly. The main purpose of Collection is,

- To listen data event and deliver it as view event.
- To fetch data from cache or SendBird server and deliver the data as view event.

To meet the purpose, each collection has event subscriber and data fetcher. Event subscriber listens data event so that it could apply data update into view, and data fetcher loads data from cache or server and sends the data to event handler.

#### ChannelCollection

Channel is quite mutable data where chat is actively going - channel's last message and unread message count may update very often. Even the position of each channel is changing drastically since many apps sort channels by the most recent message. For that reason, `ChannelCollection` depends mostly on server sync. Here's the process `ChannelCollection` synchronizes data:

1. It loads channels from cache and the view shows them for fast-loading.
2. Then it fetches the most recent channels from SendBird server and merges with the channels in view.
3. It fetches from SendBird server every time `fetch()` is called in order to view previous channels.

> Note: Channel data sync mechanism could change later.

`ChannelCollection` requires `GroupChannelListQuery` instance as it binds the query into the collection. Then the collection filters data with the query. Here's the code to create new `ChannelCollection` instance.

```java
GroupChannelListQuery query = GroupChannel.createMyGroupChannelListQuery();
// ...setup your query here

ChannelCollection collection = new ChannelCollection(query);
```

If the view is closed, which means the collection is obsolete and no longer used, remove collection explicitly.

```java
collection.remove();
```

As aforementioned, `ChannelCollection` provides event handler. Event handler is named as `ChannelCollectionHandler` and it receives `action` and `channel` list when an event has come. The `action` is a keyword to notify what happened to the channel, and the `channel` is the target `BaseChannel` instance. You can create an instance and implement the event handler and add it to the collection.

```java
ChannelCollectionHandler channelCollectionHandler = new ChannelCollectionHandler() {
    @Override
    public void onChannelEvent(final ChannelCollection collection, final List<GroupChannel> channels, final ChannelEventAction action) {
        getActivity().runOnUiThread(new Runnable() {
            @Override
            public void run() {
                switch (action) {
                    case INSERT:
                        break;
                    case UPDATE:
                        break;
                    case MOVE:
                        break;
                    case REMOVE:
                        break;
                    case CLEAR:
                        break;
                }
            }
        });
    }
};

collection.setCollectionHandler(channelCollectionHandler);

// you can cancel event subscription by calling setCollectionHandler() like:
collection.setCollectionHandler(null);
```

And data fetcher. Fetched channels would be delivered to event subscriber. Event fetcher determines the `action` automatically so you don't have to consider duplicated data in view.

```java
collection.fetch(new CompletionHandler() {
    @Override
    public void onCompleted(SendBirdException e) {
        // This callback is optional and useful to catch the moment of loading ended.
    }
});
```

#### MessageCollection

Message is relatively static data and SyncManager supports full-caching for messages. `MessageCollection` conducts background sync so that it synchronizes all the messages until it reaches to the end. Background sync does NOT affect view directly but sync local cache. For view update, explicitly call `fetch()` which fetches data from cache and sends the data into collection handler.

Background sync ceases if the sync is done or sync request is failed.

For various viewpoint support, `MessageCollection` sets starting point of view (or `viewpointTimestamp`) at creation. The `viewpointTimestamp` is a timestamp to start background sync in both previous and next direction (and also the point where a user sees at first). Here's the code to create `MessageCollection`.

```java
// customType and senderUserIds can be null.
MessageFilter filter = new MessageFilter(BaseChannel.MessageTypeFilter.ALL, customType, senderUserIds);


long viewpointTimestamp = getLastReadTimestamp(); // or Long.MAX_VALUE if you want to see the most recent messages
MessageCollection collection = new MessageCollection(groupChannel, filter, viewpointTimestamp);
```

You can dismiss collection when the collection is obsolete and no longer used.

```java
collection.remove();
```

`MessageCollection` has event handler that you can implement and add to the collection. Event handler is named as `MessageCollectionHandler` and it receives `action` and `message` list when an event has come. The `action` is a keyword to notify what happened to the channel, and the `message` is the target `BaseMessage` instance.

```java
MessageCollectionHandler messageCollectionHandler = new MessageCollectionHandler() {
    @Override
    public void onMessageEvent(MessageCollection collection, final List<BaseMessage> messages, final MessageEventAction action) {
        getActivity().runOnUiThread(new Runnable() {
            @Override
            public void run() {
                switch (action) {
                    case INSERT:
                        break;
                    case REMOVE:
                        break;
                    case UPDATE:
                        break;
                    case CLEAR:
                        break;
                }
            }
        });
    }
};
collection.setCollectionHandler(messageCollectionHandler);

// you can cancel event subscription by calling setCollectionHandler() like:
collection.setCollectionHandler(null);
```

`MessageCollection` has data fetcher by direction: `PREVIOUS` and `NEXT`. It fetches data from cache only and never request to server. If no more data is available in a certain direction, it subscribes the background sync internally and fetches the synced messages right after the sync progresses.

```java
collection.fetch(MessageCollection.Direction.PREVIOUS, new CompletionHandler() {
    @Override
    public void onCompleted(SendBirdException e) {
    }
});
collection.fetch(MessageCollection.Direction.NEXT, new CompletionHandler() {
    @Override
    public void onCompleted(SendBirdException e) {
    }
});
```

Fetched messages would be delivered to event handler. Event fetcher determines the `action` automatically so you don't have to consider duplicated data in view.

#### Handling uncaught messages

SyncManager listens message event such as `onMessageReceived` and `onMessageUpdated`, and applies the change automatically. But they would not be called if the message is sent by `currentUser`. You can keep track of the message by calling related function when the `currentUser` sends or updates message. `MessageCollection` provides methods to apply the message event to collections.

```java
// call collection.appendMessage() after sending message
UserMessage previewMessage = channel.sendUserMessage("Your Message", new BaseChannel.SendUserMessageHandler() {
    @Override
    public void onSent(UserMessage userMessage, SendBirdException e) {
        if (e != null) {
            // Error!
            collection.deleteMessage(userMessage);
            return;
        }

        // append sent message.
        collection.appendMessage(userMessage);
    }
});

collection.appendMessage(previewMessage);

// call collection.updateMessage() after updating message
channel.updateUserMessage(message.getMessageId(), "Updated message", null, null, new BaseChannel.UpdateUserMessageHandler() {
    @Override
    public void onUpdated(UserMessage userMessage, SendBirdException e) {
        if (e != null) {
            return;
        }

        collection.updateMessage(userMessage);
    }
});
```

It works only for messages sent by `currentUser` which means the message sender should be `currentUser`.

### Connection Lifecycle

You should detect connection status change and let SyncManager know the event. Call `resumeSync()` on connection, and `pauseSync()` on disconnection. Here's the code:

```java
SendBirdSyncManager.getInstance().resumeSync();
SendBirdSyncManager.getInstance().pauseSync();
```

The example below shows how to detect connection status and resume synchronization using `ConnectionHandler`. It detects disconnection automatically by `SendBird` and tries `reconnect()` internally.

```java
SendBird.addConnectionHandler(UNIQUE_HANDLER_ID, new SendBird.ConnectionHandler() {
    @Override
    public void onReconnectStarted() {
        SendBirdSyncManager.getInstance().pauseSync();
    }

    @Override
    public void onReconnectSucceeded() {
        SendBirdSyncManager.getInstance().resumeSync();
    }

    @Override
    public void onReconnectFailed() {
    }
});
```

### Cache clear

Clearing cache is necessary when a user signs out.

```java
// clear all cached data.
SendBirdSyncManager.getInstance().clearCache();

// clear specific user's cached data.
SendBirdSyncManager.getInstance().clearCache(USER_ID);
```

> WARNING! DO NOT call `SendBird.removeAllChannelHandlers()`. It does not only remove handlers you added, but also remove handlers managed by SyncManager.