---
layout: post
title:  "Communication between Web and Native"
date:   2022-05-08
categories: iOS Web Hybrid
---

This post covers how you can setup a communication channel between your WebApp and navite iOS code.

## Context
When developing for iOS and Android apps, developers often have to create the app experience separately for each platform.
Often App teams working on different platform, extract out the business logic into Web layer and use each platform as a container to deliver the app experience. In such cases, a need arises to have a bi-directional communication between Native Platform and Web to exchange messages. These types of apps provide Hybrid experience as they mix both Native and Web Technology for delivering business requirements.


## Use Case

You have a feature of your app developed using Web technology and you want to embed the same in your native Mobile application.
While the content renders inside Web, you also want to setup a communication channel between Web and your native code, so that messages can be sent from Web to Native and vice-versa for sharing data and invoking certain actions.
For example: 
- Web can request data to be downloaded by Native app and then inform web to render the data when available.
- Native can listen for notifications and inform web about it.

## Requirements
- Bi-Directional message transfer between Native and Web


## Message format
The kind of messages that is sent between Web and Native will depend on your business use case. For example, if you need to perform some action on native end, one that can not be performed in Web context, you will send a message to Native to handle it and communicate the status back to Web.
The format of message is entirely dependent on your use case. You may setup a schema contract between Web and Native and send message as per the contract.


## Implementation

The following shows a basic example of setting up the communication between Native and Web

![Native - Web Communication Overview](/assets/images/web_native_communication.png)

### 1. Create an HTML to load in WebView
Create an HTML file in your app.
e.g. `demo.html`
{% highlight html %}
<html>
<head>
        <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
        <script>
            // we'll add scripts here for Web<>Native communication
        </script>
        <body>
        <!-- we'll add content here to send message and display recieved message -->
        </body>
</html>
{% endhighlight %}

### 2. Setup HTML UI 

#### 2.1 To Post message to Native
Add a `div` inside `body` to test message from Web to Native.
- The following `div` has a `text input` where user can add a text. and a button that will trigger the message sending to native.
- The button has an action attached to it `sendMessageToNative()` which will be defined in the `script` section.

{% highlight html %}
<div>
    <h2>Message from Web</h2>
    <input type="text" id="messageOut" name="messageOut">
    <button type="button" onclick="sendMessageToNative()">Send message to Native</button>
</div>
{% endhighlight %}

#### 2.2 To receive message from Native
Add another `div` inside `body` to test message from Native to Web.
- When message is sent from Native to Web, this `div` will be updated to display the send message.

{% highlight html %}
<div>
    <h2>Message from Native</h2>
    <div id="messageIn"></div>
</div>
{% endhighlight %}

### 3. Setup HTML Script 

#### 3.1 To send message to Native
Add a JS method inside `script` to send message to Native.
- Read text from element with id `messageOut`.
- Get a messageHandler named `NativeBridge` (we'll setup `NativeBridge` later in Native code).
- post message to Native.

{% highlight javascript %}
function sendMessageToNative() {
    // Get the message from the input field
    const messageOut = document.getElementById('messageOut').value;
    
    if (!messageOut) {
        console.log('Please enter a message');
        return;
    }
    
    // Try to send message to native (iOS)
    if (window.webkit) {
        if (window.webkit.messageHandlers) {
            if (window.webkit.messageHandlers.NativeBridge) {
                window.webkit.messageHandlers.NativeBridge.postMessage(messageOut);
                console.log('Message sent to iOS native: ' + messageOut);
            } else {
                console.log('NativeBridge not found');
            }
        } else {
            console.log('No messageHandlers found');
        }
    } else {
        console.log('window.webkit not found');
    }
}
{% endhighlight %}

#### 3.2 To send message from Native
Add a JS method inside `script` to recieve message from Native.
- This method will be invoked by Native code when sending message to Web.
- It will get element by id `messageIn` and display the text send from Native code.

{% highlight javascript %}
function receiveMessageFromNative(message) {
    document.getElementById('messageIn').textContent = message;
    console.log('Message received from native: ' + message);
}
{% endhighlight %}



<br>
<hr> 
<br>

### 4 Setup Message Handler in Code

#### 4.1 Define a Message Handler
Create a string to use as the name for Message Handler
- NOTE: this is the same name that is used in HTML script.

{% highlight swift %}
enum Constant {
    static let MESSAGE_HANDLER_NAME = "NativeBridge"
}
{% endhighlight %}

#### 4.2 Create a WebView configuration
Create a `WKWebViewConfiguration` to use with the webView.

{% highlight swift %}
let configuration = WKWebViewConfiguration()
{% endhighlight %}

#### 4.3 Setup WebView process pool
Create a process pool to use with the webView.

{% highlight swift %}
let processPool = WKProcessPool()
configuration.processPool = processPool
{% endhighlight %}

#### 4.4 Create user content controller
Create an instance of `WKUserContentController` that provides a bridge between your app and the JavaScript code running in the web view.

{% highlight swift %}
let userContentController = WKUserContentController()
{% endhighlight %}

##### 4.5 Add User scripts
Add the following user scripts to the user content controller

{% highlight swift %}
let script = """
    var NativeBridge = { }
    NativeBridge.webMessageOut = function(msg) {
        window.webkit.messageHandlers.NativeBridge.postMessage(msg)
    }
    NativeBridge.webMessageIn = function(msg) {
        receiveMessageFromNative(msg)
    }
    """
let userScript = WKUserScript(
    source: script,
    injectionTime: .atDocumentStart,
    forMainFrameOnly: false
)
userContentController.addUserScript(userScript)
{% endhighlight %}

#### 4.6 Create a Message Handler
Create an instance conforming to `WKScriptMessageHandler` instance to handle message from Web.

{% highlight swift %}
class MessageHandler: NSObject, WKScriptMessageHandler {
    weak var webView: WKWebView?
    
    func userContentController(
        _ userContentController: WKUserContentController,
        didReceive message: WKScriptMessage
    ) {
        // handle message from web
    }
}
{% endhighlight %}

##### 4.6.1 Handle message received from web

The following method is called from Web. 
- Here the `message name` is used to identify the type of message being sent from web.
- We could install multiple message handlers for different type of message being sent by web.

{% highlight swift %}
func userContentController(
    _ userContentController: WKUserContentController,
    didReceive message: WKScriptMessage
) {
    switch message.name {
    case Constant.MESSAGE_HANDLER_NAME:
        Swift.print("Recieved message", message.name, message.body)
    default:
        Swift.print("Recieved other message", message.name, message.body)
    }
}
{% endhighlight %}

##### 4.6.2 Send message to web
Add a method to send message to web

The following method creates a JS script to send message to web using the Message Handler defined earlier.

{% highlight swift %}
func sendMessageToWeb(webView: WKWebView) {
    let message = "Hello from the other side: id: \(UUID().uuidString)"
    let jsScript = "\(Constant.MESSAGE_HANDLER_NAME).webMessageIn('\(message)');"
    webView.evaluateJavaScript(jsScript) { (result, error) in
        if let error = error {
            Swift.print("Failure sending message to web: \(error)")
        } else {
            Swift.print("Success: Message sent to web: \(message)")
        }
    }
}
{% endhighlight %}

#### 4.7 Add a Message Handler
Add a `WKScriptMessageHandler` instance to handle message from Web.
- create an instance of `MessageHandler`
- add the instance to userContentController for the message handler name we defined earlier.

{% highlight swift %}
let messageHandler = MessageHandler()
// store messageHandler to avoid deallocation
userContentController.add(messageHandler, name: Constant.MESSAGE_HANDLER_NAME)
{% endhighlight %}

#### 4.7 Setup user content controller

{% highlight swift %}
configuration.userContentController = userContentController
{% endhighlight %}

#### 4.8 Setup Website Data store

{% highlight swift %}
let websiteDataStore = WKWebsiteDataStore.default()

// inject any cookies that you may want
// let cookie = HTTPCookie.cookies(withResponseHeaderFields: reponseHeader, for: requestURL)
// websiteDataStore.httpCookieStore.setCookie(cookie)

configuration.websiteDataStore = websiteDataStore
{% endhighlight %}

#### 4.9 Create WebView

{% highlight swift %}
let webView = WKWebView(
    frame: CGRect(
        origin: CGPoint(x: 0, y: 0),
        size: CGSize(width: 1, height: 1)
    ),
    configuration: configuration
)

#if DEBUG
// enable inspection in DEBUG build
webView.isInspectable = true
#endif

// setup a navigation delegate
webView.navigationDelegate = navigationDelegate // assign an instance conforming to WKNavigationDelegate
{% endhighlight %}

#### 4.9 Load HTML in WebView
Attach the webView created earlier on your view and lod the HTML inside the webView.
The following  will display the `demo.html`

{% highlight swift %}
if let url = Bundle.main.url(forResource: "demo", withExtension: "html") {
    let request = URLRequest(url: url)
    webView.load(request)
}
{% endhighlight %}

Additionally create a control in your View to trigger a message send from Native to web
{% highlight swift %}
@IBAction func sendMessageToWeb(_ sender: Any) {
    if let webView = self.webView {
        messageHandler?.sendMessageToWeb(webView: webView)
    } else {
        Swift.print("Failed to send message to Web. Web view is nil")
    }
}
{% endhighlight %}

<br>
<hr>
<br>

The final HTML file will look like this:

<details>
<summary><strong>demo.html</strong></summary>
{% highlight html %}
<html>
    <head>
        <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
        <script>
            function sendMessageToNative() {
                // Get the message from the input field
                const messageOut = document.getElementById('messageOut').value;
                
                if (!messageOut) {
                    console.log('Please enter a message');
                    return;
                }
                
                // Try to send message to native (iOS)
                if (window.webkit) {
                    if (window.webkit.messageHandlers) {
                        if (window.webkit.messageHandlers.NativeBridge) {
                            window.webkit.messageHandlers.NativeBridge.postMessage(messageOut);
                            console.log('Message sent to iOS native: ' + messageOut);
                        } else {
                            console.log('NativeBridge not found');
                        }
                    } else {
                        console.log('No messageHandlers found');
                    }
                } else {
                    console.log('window.webkit not found');
                }
            }
            
            // Function to receive messages from native
            function receiveMessageFromNative(message) {
                document.getElementById('messageIn').textContent = message;
                console.log('Message received from native: ' + message);
            }
        </script>
    </head>
    <body>
        <p>Sample Web Message Exchange</p>
        
        <div>
            <h2>Message from Web</h2>
            <input type="text" id="messageOut" name="messageOut">
            <button type="button" onclick="sendMessageToNative()">Send message to Native</button>
        </div>
        </br>
        <div>
            <h2>Message from Native</h2>
            <div id="messageIn"></div>
        </div>
    </body>
</html>
{% endhighlight %}
</details>

<br>

The final Native Code setup looks like this:
<details>
<summary><strong>ViewController.swift</strong></summary>
{% highlight swift %}
import UIKit
import WebKit

extension UIView {
    func attach(child: UIView) {
        child.translatesAutoresizingMaskIntoConstraints = false
        self.addSubview(child)
        
        let top = self.topAnchor.constraint(equalTo: child.topAnchor)
        let bottom = self.bottomAnchor.constraint(equalTo: child.bottomAnchor)
        let left = self.leftAnchor.constraint(equalTo: child.leftAnchor)
        let right = self.rightAnchor.constraint(equalTo: child.rightAnchor)

        NSLayoutConstraint.activate([top, bottom, left, right])
    }
}

class ViewController: UIViewController {

    @IBOutlet weak var webViewContainer: UIView!
    @IBOutlet weak var textView: UITextView!
    
    // message handler for web message
    var messageHandler: MessageHandler?
    
    // navigation delegate
    var navigationDelegate: NavigationDelegate?
    
    var webView: WKWebView? {
        didSet {
            if let view = webView {
                webViewContainer.attach(child: view)
            } else {
                oldValue?.removeFromSuperview()
            }
        }
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        setupWebView()
    }
    
    @IBAction func sendMessageToWeb(_ sender: Any) {
        if let webView = self.webView {
            messageHandler?.sendMessageToWeb(webView: webView)
        } else {
            Swift.print("Failed to send message to Web. Web view is nil")
        }
    }
    
    
    // MARK: - setup
    
    func setupWebView() {
        // MARK: create a configuration to use with web view
        let configuration = WKWebViewConfiguration()
        
        // MARK: setup a process pool to use with web view
        let processPool = WKProcessPool()
        configuration.processPool = processPool
        
        // MARK: setup user content controller
        let userContentController = WKUserContentController()
        
        // MARK: - add required scripts to user content controller
        let script = """
            var NativeBridge = { }
            NativeBridge.webMessageOut = function(msg) {
                window.webkit.messageHandlers.NativeBridge.postMessage(msg)
            }
            NativeBridge.webMessageIn = function(msg) {
                receiveMessageFromNative(msg)
            }
            """
        let userScript = WKUserScript(
            source: script,
            injectionTime: .atDocumentStart,
            forMainFrameOnly: false
        )
        userContentController.addUserScript(userScript)
        
        // MARK: - add message handler object with a unique name
        let messageHandler = MessageHandler()
        self.messageHandler = messageHandler
        userContentController.add(messageHandler, name: Constant.MESSAGE_HANDLER_NAME)
        
        configuration.userContentController = userContentController
        
        // MARK: - setup data store
        let websiteDataStore = WKWebsiteDataStore.default()

        // inject any cookies that you may want
        // let cookie = HTTPCookie.cookies(withResponseHeaderFields: reponseHeader, for: requestURL)
        // websiteDataStore.httpCookieStore.setCookie(cookie)

        configuration.websiteDataStore = websiteDataStore
        
        // MARK: create a webView
        let webView = WKWebView(
            frame: CGRect(
                origin: CGPoint(x: 0, y: 0),
                size: CGSize(width: 1, height: 1)
            ),
            configuration: configuration
        )
        #if DEBUG
        webView.isInspectable = true
        #endif
        
        // MARK: setup navigation delegate
        let navigationDelegate = NavigationDelegate()
        webView.navigationDelegate = navigationDelegate
        
        self.webView = webView
        
        // MARK: - Load HTML
        if let url = Bundle.main.url(forResource: "demo", withExtension: "html") {
            let request = URLRequest(url: url)
            webView.load(request)
        }
    }
}
{% endhighlight %}
</details>

<details>
<summary><strong>MessageHandler.swift</strong></summary>
{% highlight swift %}
class MessageHandler: NSObject, WKScriptMessageHandler {
    weak var webView: WKWebView?
    
    func userContentController(
        _ userContentController: WKUserContentController,
        didReceive message: WKScriptMessage
    ) {
        switch message.name {
        case Constant.MESSAGE_HANDLER_NAME:
            Swift.print("Recieved message", message.name, message.body)
        default:
            Swift.print("Recieved other message", message.name, message.body)
        }
    }
    
    func sendMessageToWeb(webView: WKWebView) {
        let message = "Hello from the other side: id: \(UUID().uuidString)"
        let jsScript = "\(Constant.MESSAGE_HANDLER_NAME).webMessageIn('\(message)');"
        webView.evaluateJavaScript(jsScript) { (result, error) in
            if let error = error {
                Swift.print("Failure sending message to web: \(error)")
            } else {
                Swift.print("Success: Message sent to web: \(message)")
            }
        }
    }
}
{% endhighlight %}
</details>

<details>
<summary><strong>NavigationDelegate.swift</strong></summary>
{% highlight swift %}
// this will be covered in a separate post
class NavigationDelegate: NSObject, WKNavigationDelegate {
    /*
    public func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping @MainActor (WKNavigationActionPolicy) -> Void) {
        Swift.print(#function)
    }
    
    public func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, preferences: WKWebpagePreferences, decisionHandler: @escaping @MainActor (WKNavigationActionPolicy, WKWebpagePreferences) -> Void) {
        Swift.print(#function)
    }
    
    public func webView(_ webView: WKWebView, decidePolicyFor navigationResponse: WKNavigationResponse, decisionHandler: @escaping @MainActor (WKNavigationResponsePolicy) -> Void) {
        //
    }
    
    public func webView(_ webView: WKWebView, didStartProvisionalNavigation navigation: WKNavigation!) {
        //
    }
    
    public func webView(_ webView: WKWebView, didReceiveServerRedirectForProvisionalNavigation navigation: WKNavigation!) {
        //
    }
    
    public func webView(_ webView: WKWebView, didFailProvisionalNavigation navigation: WKNavigation!, withError error: any Error) {
        //
    }
    
    public func webView(_ webView: WKWebView, didCommit navigation: WKNavigation!) {
        //
    }
    
    public func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        //
    }
    
    public func webView(_ webView: WKWebView, didFail navigation: WKNavigation!, withError error: any Error) {
        //
    }
    
    public func webView(_ webView: WKWebView, didReceive challenge: URLAuthenticationChallenge, completionHandler: @escaping @MainActor (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        //
    }
    
    public func webView(_ webView: WKWebView, respondTo challenge: URLAuthenticationChallenge) async -> (URLSession.AuthChallengeDisposition, URLCredential?) {
        //
    }
    
    public func webViewWebContentProcessDidTerminate(_ webView: WKWebView) {
        //
    }
    
    public func webView(_ webView: WKWebView, authenticationChallenge challenge: URLAuthenticationChallenge, shouldAllowDeprecatedTLS decisionHandler: @escaping @MainActor (Bool) -> Void) {
        //
    }
    
    public func webView(_ webView: WKWebView, shouldAllowDeprecatedTLSFor challenge: URLAuthenticationChallenge) async -> Bool {
        //
    }
    
    public func webView(_ webView: WKWebView, navigationAction: WKNavigationAction, didBecome download: WKDownload) {
        //
    }
    
    public func webView(_ webView: WKWebView, navigationResponse: WKNavigationResponse, didBecome download: WKDownload) {
        //
    }
    
    public func webView(_ webView: WKWebView, shouldGoTo backForwardListItem: WKBackForwardListItem, willUseInstantBack: Bool) async -> Bool {
        //
    }
     */
}
{% endhighlight %}
</details>

<br>

![Native - Web Communication Demo setup](/assets/images/native_web_demo_setup.png)