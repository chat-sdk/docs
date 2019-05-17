## UI Customization

You can customize the UI by subclassing UI elements and then injecting them into the framework. 

### Activity Customization

For example, to override the chat activity you would subclass the `ChatActivity` then use:

```Java
ChatSDK.ui().setChatActivity(YourCustomChatActivity.class);
```

There is a method available for each of the activities in the app.

### Tab Customization

Add a tab:

```Java
ChatSDK.ui().addTab(title, iconResourceId, fragment, index);
```

Remove a tab:

```Java
ChatSDK.ui().removeTab(index);
```

To modify a tab, first make a subclass for example, you could subclass the `PrivateThreadsFragment`. Then register it with the framework:

```Java
ChatSDK.ui().setPrivateThreadsFragment(new YourFragmentSubclass());
```

### Message Customization

To customize a message you will need to make two new classes:

`YourCustomMessageDisplayHandler extends AbstractMessageDisplayHandler`

`YourCustomMessageViewHolder extends BaseMessageViewHolder`

In your new display handler, override the following method:

```Java
@Override
public AbstractMessageViewHolder newViewHolder(boolean isReply, Activity activity, PublishSubject<List<MessageAction>> actionPublishSubject) {
    View row = row(isReply, activity);
    return new YourCustomMessageViewHolder(row, activity, actionPublishSubject);
}
```

Then register the display handler with the framework for a given message type:

```Java
ChatSDK.ui().setMessageHandler(new TestTextMessageDisplayHandler(), new MessageType(MessageType.Text));
```

When the framework needs to render a new message cell, it will ask the display handler to provide it with a view and a view holder. By overriding this method, you can specify a custom view holder and modify the message rending.

#### Specify custom message XML layout

To do this override the `row` function in the display handler:

```Java
protected View row (boolean isReply, Activity activity) {
    View row;
    LayoutInflater inflater = LayoutInflater.from(activity);
    if(isReply) {
        row = inflater.inflate(R.layout.view_message_reply, null);
    } else {
        row = inflater.inflate(R.layout.view_message_me, null);
    }
    return row;
}
```  

You can return custom layout files here. Bear in mind that if you are extending the base view holder, some fields will be expected to be present. 

#### Customize the name of the message type

Override the `displayName` method in the display handler:

```Java
@Override
public String displayName(Message message) {
    return message.getText();
}
```

#### Customize the message cell

You can do this by overriding the `setMessage` function in your custom view holder. 

**Customize the message date**

```Java
@Override
public void setMessage(Message message) {
    super.setMessage(message);

    String time = String.valueOf(getTimeFormat(message).format(message.getDate().toDate()));
    timeTextView.setText(time);
}

```

Here you could also add the following calls to the `setMessage` function to customize the way the message is displayed:

**Show / Hide the icon view**

```Java
setIconHidden(true);
```

**Show / Hide the message bubble**

```Java
setBubbleHidden(false);
```

**Show / Hide change the message bubble color**

```Java        
messageBubble.getBackground()
	.setColorFilter(Color.parseColor("#b0cfea"), PorterDuff.Mode.MULTIPLY);
```

etc...