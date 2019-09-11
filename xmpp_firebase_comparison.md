## XMPP

Pros:

+ Scales to millions of concurrent connections
+ Cheaper per message at large scale
+ Works with self hosted servers
+ Full control over the data
+ Works in China
+ No internet connection required
+ Compatible with third party XMPP clients
+ Compatible with existing corporate XMPP servers

Cons:

+ Needs administrator to setup and maintain server

## Firebase

Pros: 

+ Quick and very simple to setup 
+ No server administration needed
+ Scales to 100k concurrent connections
+ Server functionality easy to modify 

Cons:

+ More expensive per message at large scale
+ Doesn't work in China
+ Internet connection required

## Summary

Firebase is great for small to medium sized apps which will not be operating in China. It's highly customizable and doesn't require many technical skills to setup or maintain. 

XMPP is better for larger installations where there could be more than 100k concurrent users. It's also cheaper to run but does require someone with XMPP server experience to setup and maintain the server. It's also better when it's necessary to interact with existing corporate XMPP servers, situations where the internet isn't available or in China where Google services are blocked. 