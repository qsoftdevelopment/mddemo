# README

## Installation

- Clone the repository (sample app included)
- Go to Android Studio with the app opened, which you want to include ChatEngine in
- File -> New -> Import Module...
- Click the `...` button (Source directory)
- Navigate to `pubnub-chat-engine-liquidcore/chatEngineLibrary/`
- Leave the Module name as `:chatEngineLibrary` and click `Finish`
- Navigate to your top-level `settings.gradle` file to have the module included
- Add `chatEngineLibrary` to the module list, so that you end up with:
  - `include ':app', 'chatEngineLibrary'`
- Navigate to your app-level `build.gradle` and add the following line
  - `implementation project(':chatEngineLibrary')`
- Congrats, you have now ChatEngine included in your Android app!

## Usage

### Initialize

- Replace the `pub_key` and `sub_key` values inside the `/values/const.xml` file with your own keys. If you're new to ChatEngine, you can get your project configured automagically [here](https://www.pubnub.com/docs/chat-engine/getting-started#automagic-pubnub-setup). Otherwise, find your keys on the _PubNub Admin Dashboard_ [here](https://admin.pubnub.com).

- Initialize the `ChatEngine` instance in the `App#onCreate()` method like this:
  - `ChatEngineSdk.initialize(this);`
- Connect to ChatEngine with the `ChatEngine#connect()` method. You should do this in a background thread. You have to connect only once at app start.
    ``` java
    ChatEngineSdk.get().connect(new ConnectListener() {
        @Override
        public void onConnected() {
            // all set!
        }

        @Override
        public void onError() {

        }

        @Override
        public void onExit(int exitCode) {

        }
    });
    ```
- Attach an instance of `EventListener` to `ChatEngine`. This callback is the key in communication with ChatEngine. All possible kinds of events will arrive in this callback.
    ``` java
    ChatEngineSdk.get().attachEventListener(new EventListener() {
        @Override
        public void onDataReceived(JSONObject jsonObject, String eventName) {
            // example
            switch (eventName) {
                case Constants.CE_INITED:
                    // initialized
                    break;
                case Constants.MESSAGE_RECEIVED:
                    // new message
                    break;
                case Constants.CHANNEL_LEFT:
                    // channel left
                    break;
                }
            }
        });
    ```

### Connect

- Connect to ChatEngine server's with your publish and subscribe keys:

    ``` java
    InitCeParam initCeParam = InitCeParam.newBuilder()
        .pubKey(Constants.PUB_KEY)
        .subKey(Constants.SUB_KEY)
        .build();
    ChatEngineSdk.get().init(initCeParam);
    ```

- You'll receive an `Constants.CE_INITED` upon successful connection.
- Next, provide your `UUID`:
  
    ``` java
    ConnectCeParam connectCeParam = ConnectCeParam.newBuilder()
        .user(new ConnectCeParam.User("john_appleseed"))
        .build();
    ChatEngineSdk.get().connect(connectCeParam);
    ```

- You'll receive an `Constants.CE_CONNECTED` event, upon successful connection.

Additionally, the `InitCeParam` builder is configurable to support common properties as seen in PubNub, like `cipherKey`, `authKey`, `presenceTimeout`, among others.

## Chat Types

The sample app included contains examples for using four different types of chat.

Instances of all types of chats are created in the same fashion:

``` java
Chat chat = Chat.newBuilder()
    .chatname("my_channel")
    .type(ChatType.GLOBAL) // or one of the possible values from ChatType
    .build();
CreateChatParams createChatParams = CreateChatParams.newBuilder()
    .uuid("other_users_uuid") // optional, pass for private, direct and feed chats
    .chat(chat)
    .build();
ChatEngineSdk.get().createChat(createChatParams);
```

Upon a successful chat creation, a `Constants.CHAT_CREATED` event will be fired and receieved in the `onDataReceived(...)` callback inside the attached `EventListener` instance, providing the following uniform payload: `{"channel":"chat-engine#chat#public.#office"}`. That's a response for a channel named `office` with the `ChatType.GLOBAL` type.

Please note that from now on, for consuming any kind of ChatEngine features, you'll always use the channel name received for `Constants.CHAT_CREATED`, instead of initially provided short name: `chat-engine#chat#public.#office` instead of just `office`.  

### Global

A global Chat room that all instances of ChatEngine connect to by default. It is a convenience property that provides some handy utility. More info [here](https://www.pubnub.com/docs/chat-engine/global-chat).

### Feed

Feed is a Chat that only streams things a User does, like 'startTyping' or 'idle' events for example. Anybody can subscribe to a User's feed, but only the User can publish to it. Users will not be able to converse in this channel. More info [here](https://www.pubnub.com/docs/chat-engine/reference/me#feed).

### Direct

Direct is a private channel that anybody can publish to but only the user can subscribe to. Great for pushing notifications or inviting to other chats. Users will not be able to communicate with one another inside of this chat. More info [here](https://www.pubnub.com/docs/chat-engine/reference/me#direct).

### Private

A user may want to make a private chat no other users can access. More info [here](https://www.pubnub.com/docs/chat-engine/private-chat).

## Send and receive messages

Regardless of the given chat type, messages are sent in the same way:

``` java
SendEventParam.Builder eventBuilder = SendEventParam.newBuilder()
    .channel(channelName)
    .eventName(EventType.MESSAGE)
    .data(text) // the actual message
    .chatType(chatType);

if (chatType.equals(ChatType.PRIVATE)) {
    eventBuilder.uuid(channelName);
}

ChatEngineSdk.get().sendEvent(eventBuilder.build());
```

After a message is sent successfully, the `Constants.MESSAGE_RECEIVED` is triggered. A perfect place for updating your messages list UI, if any.

## Chat history

A history of previusly sent messages for a given channel can be fetched like this:

``` java
ChannelHistoryParam channelHistoryParam = ChannelHistoryParam.newBuilder()
    .withChannelName(channelName)
    .withCount(20)
    .withReverse(false)
    .withStartTimeToken(startTimeToken)
    .build();
ChatEngineSdk.get().fetchChannelHistory(channelHistoryParam);
```

The counterpart event is `Constants.CHANNEL_HISTORY_RECEIVED`. Just like `Constants.MESSAGE_RECEIVED`, that's a perfect place to update the messages list UI, if any.

Please note that different chat types treat history differently, which means that the history response could possibly be empty. Also requries the _Storage & Playback addon_.

## Online users

Some chats expose an API method which returns a list of current users in that chat.

``` java
GetOnlineUsersParam getOnlineUsersParam = GetOnlineUsersParam.newBuilder()
    .withChannel("my_channel")
    .build();
ChatEngineSdk.get().getOnlineUsers(getOnlineUsersParam);
```

The `Constants.ONLINE_LIST_RECEIVED` will be fired afterwards, with this example payload: `{"users":["ron","jim","ashley"]}`.

## Online & offline events

Some types of chats have the ability to track presence states for their joined users (i.e. when they join or leave the chat).
Those events are dispatched by default for `Constants.USER_ONLINE` and `Constants.USER_OFFLINE` events.

Payloads for both events have the same structure, e.g. `{"users":[{"uuid":"jake","state":{}},{"uuid":"rob","state":"commuting"}]}`.

## Status (state)

Great for assigning other information to users.

``` java
UpdateStatusParam updateStatusParam = UpdateStatusParam.newBuilder()
    .withStateName("status") // key
    .withStateValue(statusMessage) // actual status value
    .build();
ChatEngineSdk.get().updateStatus(updateStatusParam);
```

The counterpart key is `Constants.STATUS_UPDATED` with this simple payload: `{"stateName":"status","stateValue":"in a meeting"}`

## Leave channel

Users can leave channels in two different ways:

- implicit (they are unsubscribed automatically after a predefined timeout)
- explicit (leave a channel intentionally)

An explicit channel leave can be invoked like in the following example:

``` java
LeaveChannelParam leaveChannelParam = LeaveChannelParam.newBuilder()
    .withChannel("my_channel")
    .build();
ChatEngineSdk.get().leaveChannel(leaveChannelParam);
```

The `Constants.CHANNEL_LEFT` event is triggered with the following payload: `{"channel":"chat-engine#chat#public.#office"}`.
