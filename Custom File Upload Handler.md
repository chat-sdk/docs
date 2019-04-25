# Custom file upload handler

In certain circumstances the Chat SDK needs to be able to upload files to an external server. Those situations are the following:

1. Uploading the user's profile image
2. Uploading thread images
3. Sending image messages

File upload is decoupled from the main project and is handled by the `UploadHandler` service. 

In order to customize the upload behavour or upload files to a custom server, it's necessary to create a new upload handler service and then register it with the Chat SDK framerwork. 

## The Upload Handler

To make a new upload handler you will need to subclass the `AbstractUploadHandler` for Android or the `BAbstractUploadHandler` for iOS. 

The upload handler only has one method that needs to be implemented:

Android:

```Java
Observable<FileUploadResult> uploadFile(byte[] data, String name, String mimeType);
```

iOS: 

```ObjC
-(RXPromise *) uploadFile:(NSData *)file withName: (NSString *) name mimeType: (NSString *) mimeType;
```

### iOS

To create a custom upload handler, you need to do the following:

1. Create a new subclass of the `BAbstractUploadHandler` class.
2. Implement the following method:

   ```ObjC
	-(RXPromise *) uploadFile:(NSData *)file withName: (NSString *) name mimeType: (NSString *) mimeType;
   ```
   
3. Register your new upload handler with the framework. In your App Delegate after you have called `BChatSDK.initialize` add the following line of code:

   ```ObjC
   BChatSDK.shared.networkAdapter.upload
 = [[YourUploadHandler alloc] init]
   ```

To see an example of how to implement the function, it's recommended to look at the `BFirebaseUploadHandler` class which demonstrates a model implementation for Google File Storage. 

### Android

To create a custom upload handler, you need to do the following:

1. Create a new subclass of the `AbstractUploadHandler` class.
2. Implement the following method:

   ```Java
   Observable<FileUploadResult> uploadFile(byte[] data,    String name, String mimeType);
   ```

3. Register your new upload handler with the framework. After you have called `ChatSDK.initialize` add the following line of code:

   ```Java
   ChatSDK.shared().a().upload
 = [[YourUploadHandler alloc] init]
   ```

To see an example of how to implement the function, it's recommended to look at the `FirebaseUploadHandler` class which demonstrates a model implementation for Google File Storage. 


