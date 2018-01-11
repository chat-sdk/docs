## Chat SDK Development Guide

### Contents

1. [Interacting with the server](https://github.com/chat-sdk/docs#interacting-with-the-server) - how to make requests and receive responses
2. [Handing entities](https://github.com/chat-sdk/docs#handling-entities) - Creating, modifying and deleting entities
3. [Code Examples](https://github.com/chat-sdk/docs#code-examples) - How to perform common tasks
4. [UI Customization](https://github.com/chat-sdk/docs#customizing-the-user-interface) - How to customize the UI and network interactions

### Architecture and getting started

The easiest way to get started is by understanding the core principles that are used in Chat SDK. Once you understand these principles and design patterns, it will make customization much easier. 

#### High level Architecture

The Chat SDK is broken down into the following major parts:

1. **Core:** This includes interface definitions and common services. 
2. **CoreData:** This contains the ORM which stores all the user, thread and message data
3. **UI:** This component contains all the app's user interface
4. **NetworkAdapter:** This component handles communication with the network.

Now that you know the basic structure, we're going to go into some detail about some important classes and design patterns that can be used to manipulate the Chat SDK. 

## Interacting With The Server

In this section an explanation will be provide of how to interact with the messaging server. Performing tasks like creating threads, sending messages, working with users. 

### Core Concepts

The client server interaction generally looks like this:

**The user performs some action** -> **A request is made to the server** -> **The server responds** -> **The UI is updated**

> For example, if a user writes a message and clicks "send", the message needs to be sent to the server. Once that's done, it needs to be displayed in the chat view. 

To perform these actions, you need to understand the Chat SDK service architecture. 

1. **NetworkManager:** A singleton that makes the services available to the whole app
2. **NetworkAdapter:** A wrapper class that contains references to all the possible services
3. **Handler:** A service that contains a group of related functions
4. **Function:** An individual action that can be performed.

This can be illustrated with some simple examples:

#### Creating a public thread

To create a public thread, the UI calls the following:

*iOS*

```
[[BNetworkManager sharedManager].a.publicThread createPublicThreadWithName: name]
```

>Here we have: Network Manager -> Adapter -> Handler -> Function

Or a more concise form:

```
[NM.publicThread createPublicThreadWithName: name]
```

*Android*

```
NetworkManager.shared().a.publicThread().createPublicThreadWithName(threadName)
```

>Here we have: Network Manager -> Adapter -> Handler -> Function

Or the concise form:

```
NM.publicThread().createPublicThreadWithName(threadName)
```

>**Note:**
>The NM class is just a convenience class that contains static getter functions to make calls to the NetworkManager more concise. The NM class should always be used unless you want to **set** a new handler. 

### Takeaway

The most important point is that if you want to find out which services are available, you should start by looking at the handler classes. These are documented and their names give you a good idea as to what they do. 

You can find a full list of handler classes in the `BNetworkFacade` protocol for iOS and the `BaseNetworkAdapter` class for Android. 

>**Case Study**  
>Imagine you wanted to find out how to send an image message. First you would look at the `BNetworkFacade` or the `BaseNetworkAdapter` and you would see the following property `ImageMessageHandler`. If you open that interface you would see the function `sendMessageWithImage`. Calling this would cause the image to be uploaded to the server and then the image message would be added to the thread. 

### Handling the response

After we've made a request to the server, we will need to wait some time for the server to respond. Generally speaking there are two ways to handle the response:

1. Using the **Promise** or **Observable** returned by the function
2. Listening for app level notifications

#### Promises and Observables

This document won't go into a full explanation of promises and observables because they are very common design patterns and there are plenty of excellent explanations available online. The basic idea is that the function will return an object which will allow you to register a callback to receive a notification when the function server response comes back.

>**Note**  
>A common mistake when using Observables is to forget to call `subscribe()`. Unless you call `subscribe()`, the method won't actually be executed! For example, `pushUser()` will do nothing. You have to call `pushUser().subscribe()` and then the method will be executed.  

In our example of creating a public thread:

*iOS*

```
[NM.publicThread createPublicThreadWithName:name].thenOnMain(^id(id<PThread> thread) {
    // Success
    return Nil;
}, ^id(NSError * error) {
	// Failure
    return error;
});
```

*Android*

```
NM.publicThread().createPublicThreadWithName(threadName)
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new BiConsumer<Thread, Throwable>() {
    @Override
    public void accept(Thread thread, Throwable throwable) throws Exception {
        if(throwable == null) {
          // Success
        }
        else {
          // Handle error
	    }
});
```

In each example, we can define a function that should be called if the request is successful and another that will be called if there is an error. 

>**Note**  
>You can use the `observeOn` function to tell the observable to execute the result on the main thread. 

#### App level events

There are some events that don't happen as a result of a function that we have called. For example, if another user sends us a message. These events are handled slightly differently in iOS and Android:

#### iOS

In iOS these events are handled by notifications. You can find a full list in the `BNetworkFacade.h` file. For example, if we wanted to receive notifications whenever a new message is received:

```
[_notificationList add:[[NSNotificationCenter defaultCenter] addObserverForName:bNotificationMessageAdded object:Nil queue:Nil usingBlock:^(NSNotification * notification) {
    dispatch_async(dispatch_get_main_queue(), ^{
        id<PMessage> messageModel = notification.userInfo[bNotificationMessageAddedKeyMessage];
    });
}]];
```

A couple of key points:

1. All notifications are dispatched on a background thread so if you want to update the UI, you will need to use GSD to run your code on the main thread. 

2. Sometimes the notification will contain a payload that's stored in the `userInfo` dictionary. In the example above, you can access the message. You can see what will be available by looking at the `BNetworkFacade`. Below the name of the notification, there will be one or more keys that can be used to get the user info objects. 

#### Android

In Android events are sent through an event bus using a `PublishSubject`. You can access the event bus using the following:

```
NM.events().source()
```

or 

```
NM.events().sourceOnMain()
```

When you subscribe to this source, you will receive a stream of `NetworkEvent` objects. You can also apply filters. For example:

```
NM.events().sourceOnMain()
    .filter(NetworkEvent.filterType(EventType.MessageAdded, EventType.ThreadReadReceiptUpdated))
    .filter(NetworkEvent.filterThreadEntityID(thread.getEntityID()))
    .subscribe(...);
```

Here we are only listening to the event types: `MessageAdded` and `ThreadReadReceiptUpdated` for a particular thread. 

To get a stream of all incoming messages, the following could be used:

```
Disposable d = NM.events().sourceOnMain()
        .filter(NetworkEvent.filterType(EventType.MessageAdded))
        .subscribe(new Consumer<NetworkEvent>() {
            @Override
            public void accept(NetworkEvent networkEvent) throws Exception {
                Message message = networkEvent.message;
            }
        });
```

To stop listening we can use the disposable:

```
d.dispose()
```

The Chat SDK also includes a helper class called `DisposableList`. You can add multiple disposables to this list and then call `list.dispose()` to dispose of them all at one time. 

>**Note:**
>It's important to dispose of all of your observables when you destroy an activity. Otherwise, the observer will persist and may try to perform actions on an activity which no longer exists. This will cause the app to crash. 

## Handling Entities

### Instant Messaging basics

In an instant messenger there are three core entities:

1. User (`BUser`, `co.chatsdk.core.dao.User`)
2. Thread (`BThread`, `co.chatsdk.core.dao.Thread`)
3. Message (`BMessage`, `co.chatsdk.core.dao.Message`)

>**Note:**
>In iOS, the entities are hidden behind protocol. For example, rather than dealing with a `BUser` object directly, we would always use the `id<PUser>` protocol. Because of platform differences, this isn't possible in Android so we use the database object directly. 

User has a many-to-many relationship with thread and thread has a one-to-many relationship with message. We will go into more detail about how to create and request these entities later in this guide. 

These entites exist both on the server and locally in the app's database. Both iOS and Android use an Object Relational Maping (ORM) to simplify data persistence. iOS uses **CoreData** and Android uses **GreenDAO**. 

Common tasks are handled by the `BStorageManager` singleton iOS and `co.chatsdk.core.session.StorageManager` singleton in Android.

### Creating a new Entity

*iOS*

```
id<PMessage> message = [[BStorageManager sharedManager].a createEntity:bMessageEntity];
```

*Android*

```
Message message = StorageManager.shared().createEntity(Message.class);
```

### Saving an entity

*iOS*

```
message.type = @(bMessageTypeText);
[[BStorageManager sharedManager].a save];
```

*Android*

```
message.setMessageType(MessageType.Text);
message.update();
```

### Fetching an entity using it's entity ID

*iOS*

```
id<PUser> user = [[BStorageManager sharedManager].a fetchEntityWithID:userEntityID withType:bUserEntity];
```

*Android*

```
User user = StorageManager.shared().fetchEntityWithEntityID(userEntityID, User.class);
```

> There is also a useful `fetchOrCreate` method which will try to fetch an entity and if it doesn't exist, return a new entity. 

### Deleting entities

*iOS*

```
id<PUser> user = [[BStorageManager sharedManager].a fetchEntityWithID:userEntityID withType:bUserEntity];
[[BStorageManager sharedManager].a deleteEntity: user]
```

*Android*

```
User user = StorageManager.shared().fetchEntityWithEntityID(userEntityID, User.class);
DaoCore.deleteEntity(user);
```

### Queries

More advanced queries are also possible. 

*iOS*

Get the current user's contacts.

```
NSPredicate * predicate = [NSPredicate predicateWithFormat:@"type = %@ AND owner = %@", @(bUserConnectionTypeContact), currentUser];
NSArray * entities = [[BStorageManager sharedManager].a fetchEntitiesWithName:bUserConnectionEntity withPredicate:predicate];
```

*Android*

```
List<ContactLink> contactLinks = DaoCore.fetchEntitiesWithProperty(ContactLink.class,
        ContactLinkDao.Properties.LinkOwnerUserDaoId, currentUser.getId());
```

For more advanced queries it's recommended to look at the documentation for CoreData and GreenDAO. 

## Code Examples

In this section concrete examples will be provided of how to perform common tasks.

### Authentication

Authentication is handled by the `BAuthenticationHandler` in iOS and the `AuthenticationHandler` in Android. 

It is very important to authenticate your user before you load up any of the Chat SDK views. Failure to do this will cause the app to crash. 

#### Authenticate a new user

To authenticate a user you need to pass an account details object to the authenticate method in the authentication handler. 

*iOS*

```
BAccountDetails * accountDetails = [BAccountDetails username: @"some.email@domain.com" password:@"some password"];
[NM.auth authenticate: accountDetails].thenOnMain(...);
```

*Android*

```
AccountDetails details = username("some.email@domain.com", "some password");
NM.auth().authenticate(details).subscribe(...);
```

#### Registering a new user

To register a new user, just use the Register type. 

*iOS*

```
BAccountDetails * accountDetails = [BAccountDetails signUp: @"Joe" password:@"Joe123"];
[NM.auth authenticate: accountDetails].thenOnMain(...);
```

*Android*

```
AccountDetails details = signUp("Joe", "Joe123");
NM.auth().authenticate(details).subscribe(...);
```

#### Authenticate using cached details

The Chat SDK will automatically cache the user's login details saving them from logging in each time the app opens. 

*iOS*

```
[NM.auth authenticateWithCachedToken].thenOnMain(^id(id success) {
    ...
    return Nil;
}, Nil);
```

*Android*

```
NM.auth().authenticateWithCachedToken().subscribe(...);
```

#### Logging out

*iOS*

```
[NM.auth logout];
```

*Android*

```
NM.auth().logout().subscribe(...);
```

#### Custom Authentication

With Firebase, you can also authenticate using a custom token that's been generated on your server. It works like this:

1. The user authenticates with your server
2. The server generates a new authentication token based on the user's unique ID
3. The token is passed back to the client and into the Chat SDK

##### Generating the token

To generate a token, you should follow the Firebase [custom authentication guide](https://firebase.google.com/docs/auth/admin/create-custom-tokens).

In PHP, an implementation may look like this:

```
// Get your service account's email address and private key from the JSON key file
$service_account_email = "abc-123@a-b-c-123.iam.gserviceaccount.com";
$private_key = "-----BEGIN PRIVATE KEY-----...";

function create_custom_token($uid, $is_premium_account) {
  global $service_account_email, $private_key;

  $now_seconds = time();
  $payload = array(
    "iss" => $service_account_email,
    "sub" => $service_account_email,
    "aud" => "https://identitytoolkit.googleapis.com/google.identity.identitytoolkit.v1.IdentityToolkit",
    "iat" => $now_seconds,
    "exp" => $now_seconds+(60*60),  // Maximum expiration time is one hour
    "uid" => $uid,
    "claims" => array(
      "premium_account" => $is_premium_account
    )
  );
  return JWT::encode($payload, $private_key, "RS256");
}
``` 

The `id` should be the `id` your server uses to identify the user who is currently logged in. This token should be passed back to the app.

##### Authenticating on the client

*iOS*

```
BAccountDetails * details = [BAccountDetails token:@"Your token"];
[NM.auth authenticate: details].thenOnMain(...);
```

*Android*

```
AccountDetails details = new AccountDetails.token("Your token");
NM.auth().authenticate(details).subscribe(...);
```

### Users

#### User meta data

The user entity is designed to be customisable. For that reason most of the user's properties are stored as key-value pairs. Some of the more common properties also have getters and setters for convenience. Custom data can be set and retrieved by doing the following: 

*iOS*

```
// Set the value
[user setMetaString:@"value" forKey:@"Key"];

// Get the value
[user metaStringForKey:@"key"];
```

*Android*

```
// Set the value
user.setMetaString("key", "value");

// Get the value
user.metaStringForKey("key");
```

When you push a user, these values will automatically be synchronized with the server and to all other devices have subscribed to that user. 

#### Pushing a user's details to the server

To synchronize the current user with the server the following method can be used:

*iOS*

```
[NM.core pushUser].thenOnMain(...);
```

*Android*

```
NM.core().pushUser().subscribe(...);
```

#### Subscribing to a user

In most cases, the Chat SDK will update the local database automatically if a user's details change on the remote server. By default, the Chat SDK will monitor the following users:

1. Contacts
2. Users who are members of our threads

In case you want to handle this manually, you can use the following methods:

*iOS*

```
[NM.core observeUser: @"entityID"]
```

*Android*

```
// Subscribe to user
NM.core().userOn(user);

// Unsubscribe to the user
NM.core().userOff(user);
```

#### Getting a user given the user's entity ID

In some cases, you may have a user's entity ID and want to access the user object. To do this, you need to use the user wrapper object. 

*iOS*
  
```
CCUserWrapper * wrapper = [CCUserWrapper userWithEntityID: userID];
[wrapper metaOn]
[wrapper onlineOn]
id<PUser> user = [wrapper model]
```
  
*Android*
  
```
UserWrapper wrapper = UserWrapper.initWithEntityId(userID);
wrapper.metaOn();
wrapper.onlineOn();
User user = wrapper.getModel();  
```
  
The user wrapper object is used to synchronize the local user object which is stored in the database with the remote user data stored in Firebase. When we call `metaOn` we are adding listeners so whenever the user's meta data changes on the server, the local database object will be automatically updated. `onlineOn` does a similar thing but with the user's presence (online/offline) state. 

### Contacts

#### Adding a contact

*iOS*

```
[NM.contact addContact:user withType:bUserConnectionTypeContact];
```

*Android*

```
NM.contact().addContact(user, ConnectionType.Contact).subscribe(...);
```

#### Getting a list of contacts

*iOS*

```
NSArray * users = [NM.contact contactsWithType:bUserConnectionTypeContact];
```

*Android*

```
List<User> users = NM.contact().contacts();
```

#### Adding a contact from a user ID

If you want to manage your contact list on your own server, you can use the following code to display these users on the contacts screen.

1. You would need to download the list of contacts from your server. The list should be composed of the entity IDs of the users. 

2. Download the user object using the entity ID. [See instructions here](https://github.com/chat-sdk/docs#getting-a-user-given-the-users-entity-id).
  
3. Add the user to contacts:

  *iOS*
  
  ```
  [NM.contact addContact: user withType: bUserConnectionTypeContact];
  ```
  
  *Android*
  
  ```
  NM.contact().addContact(user, ConnectionType.Contact);
  ```

4. We only need to do the above steps once. Once the user is added to contacts, whenever the app launches, all the necessary listeners will be added and the user will be displayed in the contacts view. 

### Threads

In the Chat SDK, a thread represents a conversation between a number of users. Threads can have different types: private, public, group etc... 

In iOS, threads are handled by the Core Handler. In Android, they are handled by the Thread Handler. 

#### Creating a private thread

To create a thread, you need a list of user entities that you want to add. The following code will create a new thread and then display it in the chat view. 

*iOS*

```
[NM.core createThreadWithUsers:@[user1, user2,...] name: @"Optional Name" threadCreated:^(NSError * error, id<PThread> thread) {
    UIViewController * cvc = [[BInterfaceManager sharedManager].a chatViewControllerWithThread:thread];
    [self.navigationController pushViewController:cvc animated:YES];
}];
```

*Android*

```
NM.thread().createThread("Optional Name", user1, user2, user3...)
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Consumer<Thread>() {
    @Override
    public void accept(@NonNull Thread thread) throws Exception {
        InterfaceManager.shared().a.startChatActivityForID(getApplicationContext(), thread.getEntityID());
    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(@NonNull Throwable throwable) throws Exception {
        // Handle error
    }
});
```

#### Adding or removing a user to a thread

*iOS*

```
// Adding users
[NM.core addUsers:@[user1, user2,...] toThread:thread].thenOnMain(...);

// Removing users
[NM.core removeUsers:@[user1, user2,...] fromThread:thread].thenOnMain(...);
```

*Android*

```
// Adding users
NM.thread().addUsersToThread(user1, user2,...).subscribe(...);

// Removing users
NM.thread().removeUsersFromThread(user1, user2,...).subscribe(...);
```

#### Creating a public thread

Public threads are visible to everyone who is logged into the app. They are more like public chat rooms. 

*iOS*

```
[NM.publicThread createPublicThreadWithName:name].thenOnMain(^id(id<PThread> thread) {
    UIViewController * vc = [[BInterfaceManager sharedManager].a chatViewControllerWithThread:thread];
    [self.navigationController pushViewController:vc animated:YES];
    return Nil;
}, ^id(NSError * error) {
    // Handle error
    return error;
});
```

*Android*

```
NM.publicThread().createPublicThreadWithName(threadName)
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new BiConsumer<Thread, Throwable>() {
    @Override
    public void accept(Thread thread, Throwable throwable) throws Exception {
        if(throwable == null) {
            InterfaceManager.shared().a.startChatActivityForID(getContext(), thread.getEntityID());
        }
        else {
          // Handle error
	    }
});
```

#### Getting a list of threads for a user

Sometimes it's useful to get a full list of threads for a particular type.

##### Public Threads

*iOS*

```
NSArray * threads = [NM.core threadsWithType:bThreadTypePublicGroup];
```

*Android*

```
List<Thread> * threads = NM.thread().getThreads(ThreadType.Public);
```

##### Private Threads

*iOS*

```
NSArray * threads = [NM.core threadsWithType:bThreadFilterPrivate];
```

*Android*

```
List<Thread> * threads = NM.thread().getThreads(ThreadType.Private);
```

### Messaging

Send a text message to a user:

_iOS_

```
[NM.core sendMessageWithText:text withThreadEntityID:_thread.entityID].thenOnMain(...);
```

_Android_

```
NM.thread().sendMessageWithText("Message Text", thread).subscribe(...);
```

## Customizing the User Interface

There are two main ways to customize the user interface. 

1. Modify the UI module directly
2. Change the UI by subclassing and using the **Interface Manager**

### Modifying the UI directly

Since the Chat SDK is open source, you could modify the user interface files directly. This has some benefits as well as some disadvantages. The main advantage is that this method is quick and doesn't require any configuration and allows you to see your changes immediately. The problem comes when it's time to upgrade. If you just replaced the UI module with the latest version from Github, all of your customizations would be lost and you would go back to the vanilla Chat SDK UI. 

The way to get around this is to fork the project using Git. You would make all of your customizations on a separate branch. When it was time to update the UI module, you would need to merge the latest version with your branch and resolve any conflicts that may have arisen. 

So here is an outline of the procedure. 

1. Create a [fork](https://github.com/chat-sdk/chat-sdk-ios#fork-destination-box) of the Chat SDK project on Github
2. Clone the fork to your computer using `git clone [link to your fork]`. You would replace the square brackets with the actual URL of your Github fork
3. Open your Podfile and include the Chat SDK as development pods

  ```
  pod "ChatSDK", :path => "[Absolute path to the ChatSDK.podspec file]"
  ```
  
  You should find where you downloaded the Chat SDK files and locate the **ChatSDK.podspec** file. Right click this file and click **Get Info**. Then click drag to highlight the path after it says **Where:**. Press **Command + C** to copy this path to the clipboard. Then replace the square brackets with the path you copied. 
4. Run `pod install`
5. Modify the Chat SDK directly - you can do this from within Xcode

#### Upgrading the Chat SDK

In the future you may want to upgrade the Chat SDK library. To do this, you need to complete the following steps:

1. Find the location where you saved the Chat SDK library 
2. Open this location in the terminal app
3. Add the original version of the Chat SDK as a remote

  ```
  git remote add chatsdk https://github.com/chat-sdk/chat-sdk-ios.git
  ```
  
4. Merge the latest version of the Chat SDK with your fork

  ```
  git pull chatsdk master
  ```
  
5. Resolve any conflicts

  ```
  git mergetool
  ```

[](#subclassing-using-the-inerface-manager)
### Using the interface manager

The second method is a little more complex to set up initially but it more robust over the longer term. This method involves subclassing the UI element that you need to modify. After you've made your changes, you need to find a way to tell the Chat SDK to use your subclass rather than the default class. That can be achieved using the `InterfaceManager`.

The `InterfaceManager` follows a similar pattern as the `NetworkManager`. It is a singleton class that has a replaceable adapter which provides methods that are used by the UI to request views. For example, when the app wants to show a user profile view, it uses the following method:

_iOS_

```
UIViewController * profileView = [[BInterfaceManager  sharedManager].a profileViewControllerWithUser:user];
```

_Android_

```
InterfaceManager.shared().a.startProfileActivity(getContext(), clickedUser.getEntityID());
```

Notice that the calling class just requests the profile view controller or activity. It has no idea what class will actually be returned. That is decided by the interface manager. 

The first step to add your own custom UI is to subclass the interface adapter. So we create a new class called `MyAppInterfaceAdapter` which inherits from the `DefaultInterfaceAdapter` in iOS and the `BaseInterfaceAdapter` in Android. 

_iOS_

```
@interface MyAppInterfaceAdapter : DefaultInterfaceAdapter { ...
```

_Android_

```
public class MyAppInterfaceAdaper extends BaseInterfaceAdapter { ...
```

Now we need to tell the Chat SDK to use our custom interface adapter. In the main app start method where you initialize the Chat SDK add the following:

_iOS_

```
[BInterfaceManager sharedManager].a = [[MyAppDefaultInterfaceAdapter alloc] init];
```

_Android_

```
InterfaceManager.shared().a = new MyAppInterfaceAdapter(context);
```

Next, we need to subclass the view we want to change. For example, If we wanted to modify the profile view, the first step would be to create a subclass. We could call this `MyAppProfileViewController` for iOS or `MyAppProfileActivity` for Android.

Finally, we need to override the method that provides this view in our interface adapter. To do that, add the following to your `MyAppInterfaceAdapter`.

_iOS_

```
-(UIViewController *) profileViewControllerWithUser: (id<PUser>) user {
    
    UIStoryboard * storyboard = [UIStoryboard storyboardWithName:@"MyAppProfile"
                                                          bundle:[NSBundle chatUIBundle]];
    
    BMyAppProfileTableViewController * controller = [storyboard instantiateInitialViewController];

    controller.user = user;
    return controller;
}
```

_Android_

```
public Class getProfileActivity() {
    return MyAppProfileActivity.class;
}
```

So now, lets see what happens. When the app requests the profile view from the **interface manager**, the **interface manager** will ask its adapter to provide the view. Since we replaced the standard adapter with a custom version, the **getProfile** method that you just created will be called. It will return your customized profile view which will be used by the Chat SDK. 

This method may seem a little more complex to setup initially, but it's more convenient in the long term. It means that you don't need to make any modifications to the Chat SDK library and updates can be installed without worrying about losing your changes. 

### Customizing message cells

Since table views work very differently in Android and iOS, it's helpful to look at each separately. 

#### iOS

The chat view uses a `UITableView` and the cell type that is used when rendering a specific message type is setup in the `registerMessageCells` method. 

```
(void) registerMessageCells {
    
    // Default message types
    
    [self.tableView registerClass:[BTextMessageCell class] forCellReuseIdentifier:@(bMessageTypeText).stringValue];
    [self.tableView registerClass:[BImageMessageCell class] forCellReuseIdentifier:@(bMessageTypeImage).stringValue];
    [self.tableView registerClass:[BLocationCell class] forCellReuseIdentifier:@(bMessageTypeLocation).stringValue];
    [self.tableView registerClass:[BSystemMessageCell class] forCellReuseIdentifier:@(bMessageTypeSystem).stringValue];
    
    // Some optional message types
    if ([delegate respondsToSelector:@selector(customCellTypes)]) {
        for (NSArray * cell in delegate.customCellTypes) {
            [self.tableView registerClass:cell.firstObject forCellReuseIdentifier:[cell.lastObject stringValue]];
        }
    }
}
```

First we setup the standard message types associating the cell class with a string identifier (in this case the integer value message type). 

Then we loop over the custom cell types. This means that if you want to register your own cell type, you would need to override the `customCellTypes` method:

```
-(NSMutableArray *) customCellTypes {
    NSMutableArray * types = [NSMutableArray arrayWithArray:[super customCellTypes]];
    
    [types addObject:@[[CustomMessageCell class], @(bMessageTypeText)]]
    
    return types;
}
```

#### Android

Currently, there isn't a way to define a completely custom message cell for Android. However, you can define a custom message handler which can modify the standard message cell view. To do this first you need to make a class that implements the `CustomMessageHandler` interface. 

```
public interface CustomMessageHandler {
    void updateMessageCellView (Message message, Object viewHolder, Context context);
}
```

Then you need to register your new class with the interface manager:

```
InterfaceManager.shared().a.addCustomMessageHandler(new YourCustomMessageHandler());
```

Then you need to implement the `updateMessageCellView` method. This method will be called for every cell and cells can be reused so if we're not careful we can run into problems. Imagine we want to add an icon to text message cells. 

If we used something like `layout.addView` to add the icon, this would add an icon to every text view. But the second time the view was displayed, it would add a second icon! And when the cell was reused for an image message, that message would also have the icon. 

To avoid this, we need to do the following:

1. Check the cell type - is it text or image? 
2. If it's a text view, check to see if we've already added an icon
3. If we haven't, add the icon
4. If it's not a text type and we have added an icon (it's been resued) remove the icon

```
    @Override
    public void updateMessageCellView(final Message message, Object viewHolder, final Context context) {
    
        if(viewHolder instanceof MessagesListAdapter.MessageViewHolder) {
        
            // Get the view holder and layout
            MessagesListAdapter.MessageViewHolder messageViewHolder = (MessagesListAdapter.MessageViewHolder) viewHolder;
            ViewGroup layout = messageViewHolder.extraLayout;
            
            CustomMessageView customView = null;

            // Loop over the child views
            for(int i = 0; i < layout.getChildCount(); i++) {
                View view = layout.getChildAt(i);
                
                // If the child view is an instance of our custom view break and save 
                // the view
                if(view instanceof CustomMessageView) {
                    customView = (CustomMessageView) view;
                    break;
                }
            }

            // If this isn't the correct message type remove the view
            if(message.getMessageType() != MessageType. ... ) {
                if(customView != null) {
                    layout.removeView(customView.getView());
                }
                return;
            }

			  // See note below
            if(customView == null) {
                customView = new CustomMessageView(context);
                layout.addView(customView);
            }
            if(customView.getView().getParent() == null) {
                layout.addView(customView.getView());
            }

            // The view has now been added so you can configure it as needed. 

        }
    }
```

> **Note**
> If you display a custom view in the cell, it's recommended to use create your custom class that extends `LinearLayout`. Then make a property called `View view`. Then add a method called `getView()` which can be called by the below code. This is useful because it makes it easier to retrieve your custom view. You can loop over the message cell layout and check if any sub view is an `instanceOf` your custom view. Then you can get the actual custom view using `getView()`. 

```
public class CustomMessageView extends LinearLayout {

    private View view;

    public VideoMessageView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        initView();
    }

    public VideoMessageView(Context context, AttributeSet attrs) {
        super(context, attrs);
        initView();
    }

    public VideoMessageView(Context context) {
        super(context);
        initView();
    }

    private void initView() {

        LayoutInflater inflater = LayoutInflater.from(getContext());
        view = inflater.inflate(R.layout.your_layout, null);
    }

    public View getView () {
        return view;
    }
}

```

## Modifying the database from your server

In some cases it may be necessary to access the Firebase database directly from your server. To do this, you can use the [Firebase Admin SDK](https://firebase.google.com/docs/auth/admin/). 



