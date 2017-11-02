## Chat SDK Development Guide

### Contents

1. [Interacting with the server](https://github.com/chat-sdk/docs#interacting-with-the-server) - how to make requests and recieve responses
2. [Handing entities](https://github.com/chat-sdk/docs#handling-entities) - Creating, modifying and deleteing entities
3. [Code Examples](https://github.com/chat-sdk/docs#code-examples) - How to perform common tasks
4. [UI Customization](https://github.com/chat-sdk/docs#customizing-the-user-interface) - How to customize the UI and network interactions

### Architecture and getting started

The easiesy way to get started is by understanding the core principles that are used in Chat SDK. Once you understand these principles and design patterns, it will make customization much easier. 

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

1. **NetworkManager:** A singleton that makes the services availilable to the whole app
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

Or a the concise form:

```
NM.publicThread().createPublicThreadWithName(threadName)
```

>**Note:**
>The NM class is just a convenience class that contains static getter functions to make calls to the NetworkManager more concise. The NM class should always be used unless you want to **set** a new handler. 

### Takeaway

The most important point is that if you want to find out which services are available, you should start by looking at the handler classes. These are documented and their names give you a good idea as to what they do. 

You can find a full list of handler classes in the `BNetworkFacade` protocol for iOS and the `BaseNetworkAdapter` class for Android. 

>**Case Study**  
>Imagine you wanted to find out how to send an image message. First you would look at the `BNetworkFacade` or the `BaseNetworkAdapter` and you would see the following property `ImageMessageHandler`. If you open that interface you would see the function `sendMessageWithImage`. Calling this would cause the image to be uploaded ot the server and then the image message would be added to the thread. 

### Handling the response

After we've made a request to the server, we will need to wait some time for the server to respond. Generally speaking there are two ways to handle the response:

1. Using the **Promise** or **Observable** returned by the function
2. Listening for app level notifications

#### Promises and Observables

This document won't go into a full explanation of promises and observables because they are very common design patterns and there are plenty of excellent explantions available online. The basic idea is that the function will return an object which will allow you to register a callback to recieve a notification when the function server response comes back.

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

In iOS these events are handled by notifications. You can find a full list in the `BNetworkFacade.h` file. For example, if we wanted to recieve notifications whenever a new message is received:

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

When you subscribe to this source, you will recieve a stream of `NetworkEvent` objects. You can also apply filters. For example:

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
>It's important to dispose of all of your observables when you destroy an activity. Otherwise the observer will persist and may try to perform actions on an activity which no longer exists. This will cause the app to crash. 

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

> There is also a useful `fetchOrCreate` method which will try to fetch an entity and if it doesn't exists, return an new entity. 

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
BAccountDetails * accountDetails = [BAccountDetails username: @"Joe" password:@"Joe123"];
[NM.auth authenticate: accountDetails].thenOnMain(...);
```

*Android*

```
AccountDetails details = username("Joe", "Joe123");
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

<HERE>

##### Authenticating on the client

*iOS*

```
BAccountDetails * details = [BAccountDetails token:@"Token"];
[NM.auth authenticate: details].thenOnMain(...);
```

*Android*

```
AccountDetails details = new AccountDetails.token("Token");
NM.auth().authenticate(details).subscribe(...);
```

### Users

#### User meta data

The user entity is designed to be customisable. For that reason most of the user's properties are stored as key-value pairs. Some of the more common values also have getters and setters for convenience. Custom data can be set and retrieved by doing the following: 

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

*iOS*

```
NSArray * threads = [NM.core threadsWithType:bThreadTypePublicGroup];
```

*Android*

```
List<Thread> * threads = NM.thread().getThreads(ThreadType.Public);
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

### Subclassing using the Inerface Manager

### Using a handler

Now that you know how to define a handler, it's useful to know how to call a handler from the code. 

All the handlers can be accessed through the `BNetworkManager` singleton. For example, to access the last online handler you would use the following code: 

```ObjC
[BNetworkManager sharedManager].a.lastOnline getLastOnlineForUser: ... 
```

We include the `a` because this is accessing the network adapter. 

To send a message, you would use:

```ObjC
[BNetworkManager sharedManager].a.core sendMessage: (id<PMessage>) message
```

A good approach to learn the SDK is first to identify the handler you are interested in and then search for it in the code. There you will see how it can be used. 

## Customizing the user interface

The second method is a little more complex to setup initially but it more robust over the longer term. This method involves subclassing the UI element that you need to modify. After you've made your changes, you need to find a way to tell the Chat SDK to use your subclass rather than the default class. That can be achieved using the `InterfaceManager`.

The `InterfaceManager` follows a similar parttern to the `NetworkManager`. It is a singleton class that has a replacable adapter which provides methods that are used by the UI to request views. For example, when the app wants to show a user profile view, it uses the following method:

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

So now, lets see what happens. When the app requests the profile view from the **interface manager**, the **interface manager** will ask it's adapter to provide the view. Since we replaced the standard adapter with a custom version, the **getProfile** method that you just created will be called. It will return your customized profile view which will be used by the Chat SDK. 

This method may seem a little more complex to setup initially, but it's more convenient in the long term. It means that you don't need to make any modifications to the Chat SDK library and updates can be installed without worrying about losing your changes. 





