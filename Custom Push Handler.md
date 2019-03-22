## Custom push notification handler

The Chat SDK can operate in two modes when it comes to push notifications. 

1. **Server Triggered**  
  In this mode the server is responsible for sending a push notification when the message is received. Modules are available for most XMPP servers and a Firebase Cloud Functions script is provided for Firebase installations
2. **Client triggered**  
  In this mode the client triggers the push notification after the message has been sent to the server. The client makes a HTTP request to the push server directly to trigger the push notification
  
In this document we will only be discussing client triggered push notifications and how to build a custom push notification handler. 

### The Push Handler

Push notifications are handeled by a push handler class. To gain control over how push notifications are sent, it's necessary to write a custom push handler class and then register it with the framework. 

When developing your own custom push handler a recommended starting point would be to look at the Firebase Push Handler module that is provided with the open source project. 

The push handler needs to implement three main methods:

```
void subscribeToChannel (String channel)
void unsubscribeFromChannel (String channel)
void sendPushNotification (Map data)
```

Each user needs to be assiged a channel on the push server. Usually this is the user's ID. When the user logs in, the app will try to subscribe to this channel. When the user logs out, the user should be unsubsceribed. 

Finally, there is a method which is called when a push notification needs to be sent. 

#### iOS 

To create a custom push handler you need to do the following:

1. Create a new class that is a subclass of the `BAbstractPushHandler` class. 
2. Implement the three methods detailed above. 
   
   ```
   -(void) subscribeToPushChannel: (NSString *) channel;
   -(void) unsubscribeFromPushChannel: (NSString *) channel;
   -(void) sendPushNotification: (NSDictionary *) data;
   ```

3. If you need to customize the push payload, you can override the `-(NSDictionary *) pushDataForMessage: (id<PMessage>) message` function
4. Register your push handler. After you have called `BChatSDK.initialize` in your app delegate, add the following line of code:

   ```
   BChatSDK.shared.networkAdapter.push = [[YourPushHandler alloc] init];
   ```
   
5. Make sure client push is enabled:

   ```
   config.clientPushEnabled = YES;
   ```
   
#### Android

1. Create a new class that is a subclass of the `AbstractPushHandler` class. 
2. Implement the three methods detailed above. 
   
   ```
   void subscribeToPushChannel(String channel);
   void unsubscribeToPushChannel(String channel);
   void sendPushNotification (HashMap<String, Object> data);
   ```

3. If you need to customize the push payload, you can override the `HashMap<String, Object> pushDataForMessage(Message message)` function
4. Register your push handler. After you have called `ChatSDK.initialize` in your app delegate, add the following line of code:

   ```
   ChatSDK.shared().a().push = new YourPushHandler();
   ```
   
5. Make sure client push is enabled:

   ```
   config.setClientPushEnabled(true);
   ```