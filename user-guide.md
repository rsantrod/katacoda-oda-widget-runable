# Web SDK

Use Oracle Digital Assistant's Client SDK for JavaScript to add live messaging to your website or web app.

## Installation

Adding Web SDK to your project is easy. To get started, you need to use the Script Tag method.
1. First, save web-sdk.js in your project directory. Keep note of its location.
2. Add the following code towards the end of the head section on your HTML page.  Replace `<WebSDK URL>` with location of web-sdk.js, and `<WebSDK namespace>` with a name for the library, which is typically `Bots`. The library will be referred by that name and used to make the API calls.
```html
<script>
    var chatSettings = {
        URI: '<Server URI>',
        channelId: '<Channel ID>',
        userId: '<User ID>'
    };
    function initSdk(name) {
        // Default name is Bots
        if (!name) {
            name = 'Bots';
        }
        setTimeout(() => {
            const Bots = new WebSDK(chatSettings); // Initiate library with configuration
            Bots.connect()                         // Connect to server
                .then(() => {
                    console.log("Connection Successful");
                })
                .catch((reason) => {
                    console.log("Connection failed");
                    console.log(reason);
                });
            window[name] = Bots;
        });
    }
</script>
<script src="<WebSDK URL>" onload="initSdk('<WebSDK namespace>')">
```

If you are connecting to a channel with client authentication enabled, minor modifications are needed. Along with setting the `clientAuthEnabled` property to true, you must also include a function that generates and passes a JWT token. The function must return a Promise that resolves to a signed JWT token string. The SDK will use this function to generate a new token whenever it needs to establish a new connection and the existing token is expired. The script would look something like this:

```html
<script>
    var chatSettings = {
        URI: '<Server URI>',
        clientAuthEnabled: true
    };
    var generateToken = function() {
        return new Promise((resolve) => {
            fetch('https://yourbackend.com/endpointToGenerateJWTToken').then((token) => {
                resolve(token);
            })
        });
    }
    function initSdk(name) {
        // Default name is Bots
        if (!name) {
            name = 'Bots';
        }
        setTimeout(() => {
            const Bots = new WebSDK(chatSettings, generateToken);  // Initiate library with configuration
            Bots.connect()                                                  // Connect to server
                .then(() => {
                    console.log("Connection Successful");
                })
                .catch((reason) => {
                    console.log("Connection failed");
                    console.log(reason);
                });
            window[name] = Bots;
        });
    }
</script>
<script src="<WebSDK URL>" onload="initSdk('<WebSDK namespace>')">
```

And _voila!_ You have an app integrated with Oracle Web SDK.

### Importing SDK library in JavaScript

To import the library dynamically in JavaScript, following utility function can be used, which is based on [MDN example](https://developer.mozilla.org/en-US/docs/Web/API/HTMLScriptElement#Dynamically_importing_scripts) for importing scripts dynamically:

```js
function fetchSDK(src, onloadFunction) {
    const script = document.createElement('script');
    script.type = 'application/javascript';
    script.async = true;    // load the script asynchronously
    script.defer = true;    // fallback support for browsers that does not support async
    script.onload = onloadFunction;
    document.head.appendChild(script);
    script.src = src;
}

fetchSDK('<path of the web-sdk>', initSdk);
```

## Browser Support
The SDK is supported on these browsers on Mac, Windows, and Linux:
* Firefox
* Chrome
* Safari
* Edge

For mobile platforms the following browsers are supported:
* Android browser
* iOS Safari

Internet Explorer is **NOT** supported by the SDK.

## SDK Security

The Web SDK can connect to Digital Assistant web channel in two modes:

### Channel with Client Auth Disabled

In this case, only connections made from whitelisted domains are allowed at the server.

This use case is recommended when the client application can't generate a signed JWT Token (static website or no authentication mechanism for the web/mobile app) but wants to have ODA integration. It can also be used when the chat widget is already secured and visible only for authenticated users in the client platforms (Web Application with protected page).

When connection to such channels, the **channelId** must be passed as a setting parameter during SDK initialization. The **userId** is also required to establish the connection. If you don't provide a value during initialization, then the SDK generates a random userId..

```js
{
    URI: 'oda-instance.com',
    channelId: '626f5db1-f99a-4984-86ee-df2d734537e6',
    userId: 'Jessica'
}
```

### Channel with Client Auth Enabled

In this case, in addition to domain whitelisting, client authentication is enforced by signed JWT tokens.

The token generation and signing must be done by the client in the backend server, preferable after user/client authentication, which is capable of maintaining the keyId and keySecret safe.

When the SDK needs to establish a connection with the ODA server, it first requests a JWT token from the client and then sends it along with the connection request. The ODA server validates the token signature and obtains the claim set from the JWT payload to verify the token to establish the connection.

To enable this mode, two fields are required during SDK initialization: **clientAuthEnabled: true** must be passed in the SDK settings parameter, and a **token generator function** must be passed as the second parameter. The function must return a Promise, which is resolved to return a signed JWT token string.

```js
const generateToken = function() {
    return new Promise((resolve) => {
        fetch('https://yourbackend.com/endpointToGenerateJWTToken').then((token) => {
            resolve(token);
        });
    });
}

Bots.init({
    URI: 'oda-instance.com',
    clientAuthEnabled: true
}, generateToken);
```

### JWT Token

The JWT token generation and signing are the responsibility of the client application, some of the token payload fields are mandatory and are validated by the ODA server.

Clients must use the HS256 signing algorithm to sign the tokens. The tokens must be signed by the secret key of the client auth enabled channel to which the connection is made. The body of the token must have the following claims:

* iat - issued at time
* exp - expiry time
* channelId - channel ID
* userId - user ID

Here's a sample signed JWT token:

Encoded token:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzY3NDcyMDUsImV4cCI6MTU3Njc1MTExNywiY2hhbm5lbElkIjoiNDkyMDU5NWMtZWM3OS00NjE3LWFmOWYtNTk1MGQ2MDNmOGJiIiwidXNlcklkIjoiSm9obiIsImp0aSI6ImQzMjFjZDA2LTNhYTYtNGZlZS05NTBlLTYzZGNiNGVjODJhZCJ9.lwomlrI6XYR4nPQk0cIvyU_YMf4gYvfVpUiBjYihbUQ
```

Decoded token:

Header:

```json
{
    "typ": "JWT",
    "alg": "HS256"
}
```

Payload:

```json
{
    "iat": 1576747205,
    "exp": 1576748406,
    "channelId": "4920595c-ec79-4617-af9f-5950d603f8bb",
    "userId": "John"
}
```

Any additional claims passed in the payload do not affect the client authentication mechanism.

## Settings

Use these properties to configure the chat bot widget:

### Network Configuration
| Property name | Optional | Default Value | Description |
| ------ | ------ | ------ | ------ |
| URI | No || Chat server URL |
| channelId | No || Web Channel ID |
| userId | Yes | Random generated value | Unique identifier for user, gets initialized by SDK if not provided |
| clientAuthEnabled | Yes | `false` | Used to determine whether SDK is connecting to client authentication enabled channel. This must be set to true to connect to such channel and use jwt token for authentication. |

### Feature Flags
| Property name | Optional | Default Value | Description |
| ------ | ------ | ------ | ------ |
| displayActionsAsPills | Yes | `false` | Display action buttons as pills |
| enableAttachment | Yes | `true` | Flag to configure attachment sharing in the chat widget |
| enableAutocomplete | Yes | `false` | Flag to enable autocomplete feature when user types a message to the bot |
| enableBotAudioResponse | Yes | `false` | Flag to configure how the bot's responses are uttered. If true, the bot's responses are spoken using the Web API. |
| enableClearMessage | Yes | `false` | Flag to enable button to clear messages on the chat widget header |
| enableHeadless | Yes | `false` | Flag to use the SDK without its UI. Allows developers to develop their own UI for chat. |
| enableLocalConversationHistory | Yes | `false` | Flag to enable loading previous conversations for given userId on the given browser when widget is initialized |
| enableLongPolling | Yes | `false` | Use http requests to connect to server if WebSocket fails to connect |
| enableSecureConnection | Yes | `true` | Flag to configure secure communication - https vs http and wss vs ws |
| enableSpeech | Yes | `false` | Flag to use the speech recognition service to convert the user voice to text messages |
| enableSpeechAutoSend | Yes | `true` | Flag to enable or disable automatically sending voice recognized text to chat server |
| enableTimestamp | Yes | `true` | Flag to enable read timestamp for messages |
| openChatOnLoad | Yes | `false` | Flag to expand chat widget when page is loaded |
| openLinksInNewWindow | Yes | `false` | If true, when a link in a message is clicked, the target is opened in a new window instead of following the user's browser preference |
| showConnectionStatus | Yes | `false` | Flag to enable display of connection status in header |
| showPrevConvStatus | Yes | `true` | Flag to configure display of status message at the end of older messages from previous conversations |
| showTypingIndicator | Yes | `true` | Flag to display chat bubble when waiting for bot's response |
| disablePastActions | Yes | `'all'` | Field to disable button clicks on messages that user has interacted with. Allowed values `all`, `none`, and `postback` |

### Functionality Configuration
| Property name | Optional | Default Value | Description |
| ------ | ------ | ------ | ------ |
| initBotAudioMuted | Yes | `true` | Only applicable when `enableBotAudioResponse` is `true`. This flag determines whether the bot message utterance will initially be active or muted. By default, it is set to `true` and the utterance is muted. |
| enableAutocompleteClientCache | Yes | `false` | Flag to enable the client side cache to minimize server calls when the user uses the autocomplete feature |
| customHeaderElementId | Yes || HTML element ID of the div to add to the chat widget header. Used for adding additional elements to the chat widget header. |
| delegate | Yes || Object that enables you to set a delegate to receive callbacks before certain events in the conversation. More detail is provided later in this document. |
| embedded | Yes | `false` | Flag to activate embedded mode for the chat widget |
| targetElement | Yes (`embedded` must be true) | widgetDiv | div in which to embed the chat widget |
| embeddedVideo | Yes | `false` | If true, YouTube video links in the text messages can be played within the chat widget |
| embedTopStickyId | Yes || ID of element to be added as a pinned content at top of chat |
| embedTopScrollId | Yes || ID of element to be added as a scrolling content at top of chat |
| embedBottomScrollId | Yes || ID of element to be added as a scrolling content at bottom of chat |
| embedBottomStickyId | Yes || ID of element to be added as a pinned content at bottom of chat |
| isDebugMode | Yes | `false` | Flag to enable debug mode |
| initUserHiddenMessage | Yes || A user message to send once the chat widget is ready without displaying the message in the chat widget. Used to initiate the conversation automatically. The message can be a text string or a message object. |
| initUserProfile | Yes || Initializes the user profile before the conversation starts. The profile payload must be of format `{ profile: {...} }`. The profile is updated before sending the value in `initUserHiddenMessage`. |
| i18n | Yes | `{'en': {...}}` | Object that contains locale fields; each locale maintains i18n key-value pairs for certain strings that are used in the widget. See the Custom Text section for the strings that you can customize. |
| locale | Yes | `'en'` | The default locale which is used for the informational text in the SDK. The locale passed has higher preference over users' browser locales. If there isn't an exact match, then the SDK attempts to match the closest language. For example, if the locale is `'da-dk'`, but i18n translations are provided only for `'da'`, then the `'da'` translation is used. In absence of translations for passed locale, translations are searched and applied for browser locales. In absence of translations for any of them, the default locale, `'en'` is used for translations. |
| messageCacheSizeLimit | Yes | `2000` | Maximum number of messages to be saved in localStorage at a time |
| readMark | Yes | `'✓'` | symbol for read messages, disabled if enableTimestamp is set to false |
| skillVoices | Yes | System language | Array containing preferred voices to use for narrating responses. Each item in the array should be an object with two fields: `lang`, and `name`. `name` is optional. The first item that matches a voice that's available in the system will be used for the narration. |
| typingIndicatorTimeout | Yes | `20` | Allows setting the time in number of seconds after which the typing indicator is automatically removed if the chat widget has not yet received the response. |

### Layout Modification
| Property name | Optional | Default Value | Description |
| ------ | ------ | ------ | ------ |
| badgePosition | Yes | `{"top": "0", "right": "0"}` | Position of the notification badge icon with respect to the bot button |
| colors | Yes | `{"branding": "#1B8FD2", "text": "#212121", "textLight": "#737373"}` | Colors used in the chat widget. See the Custom Colors section for more details. |
| conversationBeginPosition | Yes | `'top'` | Starting position of the conversation in the widget. If set to `'top'`, the first messages appear on the top of widget. If `'bottom'`, they appear on the bottom. |
| font | Yes | `'16px "Helvetica Neue", Helvetica, Arial, sans-serif'` | Font for chat widget |
| height | Yes | `'620px'` | Height of chat widget. Must be set to one of [length](https://developer.mozilla.org/en-US/docs/Web/CSS/length) values |
| messagePadding | Yes | `'15px'` | Padding around messages in chat widget |
| position | Yes | `{bottom: '20px', right: '20px'}` | Field to specify where to place the chat widget in the browser window. Must be passed as a JSON object |
| width | Yes | `'375px'` | Width of chat widget. Must be set to one of [length](https://developer.mozilla.org/en-US/docs/Web/CSS/length) values |

### Custom Icons

The icons used in the widget can be customized in two ways - URL of the image asset, or an SVG string. For the icons that support SVG strings, you can pass the raw SVG data and the widget renders it as an inline SVG. SVG string allows for faster loading of image, ability to animate it, or change its color. The theme color is applied to SVG strings for attachment, send, and mic buttons, but not other image assets.

| Property name | SVG String Compatible | Description |
| ------ | ------ | ------ |
| botButtonIcon | No | Chat bot button, displayed as the minimized chat button |
| logoIcon | No | Chat logo icon, displayed at the header of the chat widget |
| botIcon | No | Icon alongside bot response messages. If not provided, no icon is displayed |
| personIcon | No | Icon alongside user messages. If not provided, no icon is displayed |
| chatBubbleIcon | Yes | Loading chat bubble icon |
| clearMessageIcon | Yes | Clear message button icon at widget header |
| audioResponseOffIcon | Yes | Icon of toggle button when audio responses are turned off |
| audioResponseOnIcon | Yes | Icon of toggle button when audio responses are turned on |
| closeIcon | Yes | Icon for button to minimize chat widget at widget header |
| attachmentIcon | Yes | Attachment upload button icon |
| sendIcon | Yes | Send message button icon |
| micIcon | Yes | Mic button icon |
| audioIcon | No | Audio attachment icon, displayed when attachment source URL is not reachable |
| fileIcon | No | File attachment icon |
| imageIcon | No | Image attachment icon, displayed when attachment source URL is not reachable |
| videoIcon | No | Video attachment icon, displayed when attachment source URL is not reachable |
| errorIcon | No | Error icon. Displayed with error messages or information  |

| Property name | Optional | Default Value | Description |
| ------ | ------ | ------ | ------ |
| chatBubbleIconHeight | Yes | `'42px'` | Height of loading chat bubble icon |
| chatBubbleIconWidth | Yes | `'56px'` | Width of loading chat bubble icon |

Here's an example of providing custom icons:
```js
const chatSettings = {
    ...,
    clearMessageIcon: 'https://cdn4.iconfinder.com/data/icons/kids-2/128/kids_kid_boy-32.png',
    micIcon: '<svg xmlns="http://www.w3.org/2000/svg" height="24" viewBox="0 0 24 24" width="24"><path d="M0 0h24v24H0z" fill="none" opacity=".1"/><path d="M12 1c-4.97 0-9 4.03-9 9v7c0 1.66 1.34 3 3 3h3v-8H5v-2c0-3.87 3.13-7 7-7s7 3.13 7 7v2h-4v8h4v1h-7v2h6c1.66 0 3-1.34 3-3V10c0-4.97-4.03-9-9-9z"/></svg>'
};
```

### Custom Colors
Following colors can be modified to provide a custom look for the widget. The color provided must be a hexadecimal color. If a color is not provided, the default is used.

| key | Default Value | Description |
| ------ | ------ | ------ |
| branding | `'#1B8FD2'` | The primary color for branding the widget. The color is used as the header background, and as hover color on footer buttons. |
| text | `'#212121'` | The text color of messages in chat widget |
| textLight | `'#737373'` | The text color of secondary text in messages, e.g. card descriptions in chat widget |
| headerBackground | Same as `branding` | The background color of chat widget's header |
| conversationBackground | `'#ffffff'` | The background of chat widget's conversation pane. It accepts all values supported by `background` [CSS property](https://developer.mozilla.org/en-US/docs/Web/CSS/background), so it can be used to set image backgrounds, gradients etc. as well. |
| cardBackground | `'#ffffff` | The background color of card messages |
| footerBackground | `'#ffffff'` | The background color of chat widget's footer |
| botMessageBackground | `'#e5f1ff'` | The background color of response messages |
| botText | Same as `text` | The text color of content in response messages |
| userMessageBackground | `'#ececec'` | The background color of user messages |
| userText | Same as `text` | The text color of user messages |
| notificationBadgeBackground | `'#e6081a'`  | The background color of message notification badge |
| notificationBadgeText | `'#ffffff'` | The text color of message count in notification badge |

Here's an example of providing custom color.
```json
"colors": {
    "branding": "#e00",
    "text": "#545454"
}
```
The branding and text color will be modified in above example. Default color is used for secondary text color.

### Custom Text
The following strings can be customized to provide localized text. The strings must be provided under a valid [language string](https://www.science.co.il/language/Locale-codes.php) e.g. 'en-us', 'es', 'fr', 'zh-cn'.

| key | Default Value | Description |
| ------ | ------ | ------ |
| audioResponseOff | `Audio response off` | Tooltip that appears when hovering over audio utterance on button in header. |
| audioResponseOn | `Audio response on` | Tooltip that appears when hovering over audio utterance off button in header. |
| card | `Card` | CRC Card identifier. |
| cardImagePlaceholder | `Loading image` | Text placeholder that is displayed while card image is fetched and loaded. |
| chatSubtitle | | Subtitle of the chat widget that is displayed below the title in the header. |
| chatTitle | `Ask` | Title of the chat widget that is displayed on the header.  |
| chatSubtitle || Subtitle of the chat widget that is displayed on the header below the title. If `showConnectionStatus` is set true, and the subtitle is set as well, the subtitle is displayed instead of the connection status. |
| clear | `Clear` | Tooltip that appears when hovering over clear messages button in header. |
| close | `Close` | Tooltip that appears when hovering over close widget button in header. |
| closing | `Closing` | Status text when connection between chat widget and server is closing. |
| connected | `Connected` | Status text when connection between chat widget and server is established. |
| connecting | `Connecting` | Status text when connection between chat widget and server is connecting. |
| disconnected | `Disconnected` | Status text when connection between chat widget and server is closed. |
| inputPlaceholder | `Type a message` | Placeholder text that appears in user input field. |
| noSpeechTimeout | `Could not detect the voice, no message sent.` | Status text that is displayed when server is unable to recognize the voice. |
| previousChats | `Previous conversations` | Status text that is displayed at the end of older messages. |
| requestLocation | `Requesting location` | Text that is displayed while user location is requested on browser. |
| requestLocationString | `Cannot access your location. Please allow access to proceed further.` | Error text that is displayed when location request is denied. |
| retryMessage | `Try again` | Text that is displayed when user message is not sent to server. |
| send | `Send` | Tooltip that appears when hovering over send button in footer. |
| speak | `Speak` | Tooltip that appears when hovering over speak button in footer. |
| upload | `Upload Document` | Tooltip that appears when hovering over upload button in footer. |
| uploadFailed | `Upload Failed.` | Error text for upload failure. |
| uploadFileSizeLimitExceeded | `Upload Failed. File size must not be more than 25MB.` | Error text displayed when upload file size is too heavy. |
| uploadUnsupportedFileType | `Upload Failed. Unsupported file type.` | Error text displayed when unsupported file type upload is attempted. |
| utteranceGeneric | `Message from skill.` | Fallback description of response message that is used in utterance. |

Here's an example of providing custom text in a few locales.
```json
"i18n": {
    "fr": {
        "chatTitle": "Soutien"
    },
    "en": {
        "chatTitle": "Support"
    },
    "es": {
        "chatTitle": "Apoyo"
    },
    "zh-cn": {
        "chatTitle": "支持"
    }
}
```
If custom strings are provided for any locale other than 'en', all keys must be specified for that locale, otherwise 'en' translations are displayed for the missing values.

The settings can be configured as provided in the example below:
```js
<script>
var chatSettings = {
    URI: '<Chat server URI>',
    channelId: '<Channel ID>',
    userId: '<User ID>',
    font: '14px "Helvetica Neue", Helvetica, Arial, sans-serif',
    locale: 'en',
    enableClearMessage: true,
    botButtonIcon : 'style/Robot.png',
    botIcon: 'style/Robot.png',
    personIcon: 'style/userIcon.png'
};
function initSdk(name) {
    // Default name is Bots
    if (!name) {
        name = 'Bots';
    }
    setTimeout(() => {
        const Bots = new WebSDK(chatSettings); // Initiate library with configuration

        // Add ready listener before calling connect()
        // It is preferable to add other listeners before calling connect() too
        Bots.on('ready', () => {
            console.log('The widget is ready');
        });

        Bots.connect(); // Connect to server
        window[name] = Bots;
    });
}
</script>
<script src="<WebSDK URL>" onload="initSdk('<WebSDK namespace>')">
```

### Customizing CSS Classes

To further customize the look of the chat widget, its CSS classes can be overridden with custom style rules. Some of the main classes are as follows:

| class | Component |
| ------ | ------ |
| oda-chat-wrapper | Wrapper for entire chat widget |
| oda-chat-button | Collapsed chat widget button |
| oda-chat-notification-badge | Unseen message notification badge |
| oda-chat-widget | Expanded chat widget; wraps the widget header, conversation, and footer |
| oda-chat-header | Chat widget header |
| oda-chat-logo | Logo on the widget header |
| oda-chat-title | Title on the widget header |
| oda-chat-connection-status | Connection status. Each connection value has its own class as well - oda-chat-connected, oda-chat-disconnected etc. |
| oda-chat-connected | Applied as a sibling to connection-status when the widget is connected to server |
| oda-chat-connecting | Applied as a sibling to connection-status when the widget is connecting to server |
| oda-chat-disconnected | Applied as a sibling to connection-status when the widget is disconnected from server |
| oda-chat-closing | Applied as a sibling to connection-status when the widget is disconnecting from server |
| oda-chat-header-button | Common class for all header buttons |
| oda-chat-button-clear | Clear messages button |
| oda-chat-button-narration | Bot audio response toggle button |
| oda-chat-button-close | Close widget button |
| oda-chat-conversation | Container for the conversation |
| oda-chat-message | Common wrapper class for all chat messages |
| oda-chat-left | Bot message wrapper |
| oda-chat-right | User message wrapper |
| oda-chat-icon-wrapper | Bot/person icon wrapper displayed alongside message |
| oda-chat-message-icon | Bot/person icon image displayed alongside message |
| oda-chat-message-bubble | Message bubble |
| oda-chat-message-actions | Action buttons wrapper |
| oda-chat-message-global-actions | Global action buttons wrapper |
| oda-chat-message-action-postback | Postback action button |
| oda-chat-message-action-location | Location request action button |
| oda-chat-card | Card message |
| oda-chat-footer | Chat widget footer |
| oda-chat-footer-button | Common class for all footer buttons |
| oda-chat-button-upload | Upload file button |
| oda-chat-button-send | Send message button |
| oda-chat-user-input | User input textarea |

## Methods

The name that's passed in the `initSdk` parameter (WebSDK Namespace) will be used to invoke the SDK's public APIs. For example, if the name is set as `'Bots'`, the APIs can be invoked with `Bots.<API>()`.


### openChat()

Opens the chat widget popup.

```js
Bots.openChat();
```

### closeChat()

Collapses the chat widget and displays the chat icon.

```js
Bots.closeChat();
```

### isChatOpened()

Returns true if the widget is open, otherwise false.

```js
Bots.isChatOpened();
```

### destroy()

Destroys the widget, making the UI element disappear.

```js
Bots.destroy();
```

### connect()

Connects to current chat server.

```js
Bots.connect()
```

The API returns a promise which can be used to execute code after connection is complete or closed.

```js
Bots.connect()
    .then(
        function() {
            // Successful connection
        },
        function(error) {
            // Something went wrong during connection
        }
    )
```

### connect(config)

Connects to current chat server with passed parameters, userId is optional.

```js
Bots.connect({
    channelId: '<channelId>',
    userId: '<userId>'})
```

The API returns a promise which can be used to execute code after connection is complete or closed.

```js
Bots.connect({
    channelId: '<channelId>',
    userId: '<userId>'})
    .then(
        function() {
            // Successful connection
        },
        function(error) {
            // Something went wrong during connection
        }
    )
```


### disconnect()

Closes currently active connection to the chat server.

```js
Bots.disconnect()
```

### isConnected()

Returns whether the a connection to the chat server is currently active.

```js
Bots.isConnected() // false
```

### sendAttachment(file)

Uploads file to the server. Returns a Promise that is resolved with the server response for successful uploads, or gets rejected with an error message for unsuccessful uploads.

```js
Bots.sendAttachment(file)
    .then((result) => {
        console.log('File uploaded successfully.');
    })
    .catch((reason) => {
        console.log(reason);
    });
```

### sendMessage(message, flags)

Sends a message on user’s behalf. Returns a boolean to indicate whether the message was sent.

```js
Bots.sendMessage({
    type: 'text',
    text: 'hello'
});
// OR
Bots.sendMessage('hello');
```

The messages can be send in three formats:

* passing text string
```js
Bots.sendMessage('Show my payslip')
```

You can also pass a location request's response in a similar manner:

```js
Bots.sendMessage('32.323232, 143.434343')   // Latitude, longitude
```

* passing messagePayload of message

```js
Bots.sendMessage({
    type: 'text',
    text: 'Show menu'
})

Bots.sendMessage({
    postback: {
        variables: {},
        "system.botId": "5409672A-AB06-4109-8303-318F5EDE91AA",
        action: "pizza",
        "system.state": "ShowMenu"
    },
    text: "Order Pizza",
    type: "postback"
})
```

* passing entire message

```js
Bots.sendMessage({
    "messagePayload": {
        "postback": {
            "variables": {},
            "system.botId": "5409672A-AB06-4109-8303-318F5EDE91AA",
            "action": "pizza",
            "system.state": "ShowMenu"
        },
        "text": "Order Pizza",
        "type": "postback"
    },
    "userId": "Guest"
})
```

To keep the message from being displayed in the widget, pass the `hidden` boolean flag in the flags object.

```js
Bots.sendMessage('What is the menu today?', { hidden: true });
```

The above call will send the message to server without displaying it in the widget.

### updateUser(userDetails)

Updates the user information.

```js
Bots.updateUser({
    messagePayload: {
        text: 'give me a pizza',
        type: 'text'
    },
    profile: {
        firstName: 'Jane',
        lastName: 'Smith',
        email: 'updated@email.com',
        properties: {
            justGotUpdated: true
        }
    }
});
```

Note: You also can use the `initUserProfile` property to set profile values. If you use the `initUserHiddenMessage` property, and the response from the server is expected to have user profile values, then you must use the `initUserProfile` property instead of setting the values in `updateUser()`. For later updates to user profile, you can continue using `updateUser()`.

### startVoiceRecording(onSpeechRecognition, onSpeechNetworkChange)

Starts voice recording if the speech recognition feature is enabled. The API returns a Promise that is resolved when the speech service is able to start recording. In case of an error, the Promise is rejected with an Error object.
The API expects two callbacks to be passed as its parameters:
`onSpeechRecognition` function is called on the speech server response. The JSON response is passed as parameter.
`onSpeechNetworkChange` function is called on change in the speech connection. Two parameters are passed - WebSocket status, and `closeEvent` in case of closed connection.

```js
Bots.startVoiceRecording((data) => {
    let recognizedText = '';
    if (data && (data.event === 'finalResult' || data.event === 'partialResult')) {
        if (data.nbest && data.nbest.length > 0) {
            recognizedText = data.nbest[0].utterance;
        }
    }
}, (status, error) => {
    if (status === WebSocket.OPEN) {
        // Connection established
    } else if (status === WebSocket.CLOSED) {
        // Connection closed
    }
})
```

### stopVoiceRecording()

Stops any running voice recording. Throws an error if speech recognition is not enabled.

```js
Bots.stopVoiceRecording()
```

### setDelegate(delegate)

The `setDelegate` method can be used to change or remove delegate callbacks after a conversation has been initialized. If no value is passed, the existing delegates are removed.

```js
Bots.connect().then(() => {
    Bots.setDelegate({
        beforeDisplay(message) {
            return message;
        },
        beforeSend(message) {
            return message;
        },
        beforePostbackSend(postback) {
            return postback;
        }
    })
});
```

### getConversationHistory()

Returns information about current conversation for the user. The response is an object containing user ID, messages, and unread message count.

```js
Bots.getConversationHistory()

{
    "messages":[
        {
            "date":"2020-01-22T17:06:24.085Z",
            "isRead":true,
            "messagePayload":{
                "text":"show menu",
                "type":"text"
            },
            "userId":"user74882436"
        }
    ],
    "messagesCount": 1,
    "unread": 0,
    "userId":"user74882436"
}
```

### getUnreadMessagesCount()

Returns the count of the unread messages.

```js
Bots.getUnreadMessagesCount();
```

### setAllMessagesAsRead()

Marks all messages as have been read/seen by user.

```js
Bots.getUnreadMessagesCount(); // 2
Bots.setAllMessagesAsRead();
Bots.getUnreadMessagesCount(); // 0
```

### setUserInputMessage(text)

Fills user input field with passed message.

```js
Bots.setUserInputMessage('Show my agenda for today please.');
```

### setUserInputPlaceholder(text)

Updates user input's placeholder text. It can be utilized to change the placeholder text according to the input expected from the user.

```js
Bots.setUserInputPlaceholder('Add your pizza topping');
```

### setChatBubbleIconHeight(height)

Sets the height of loading chat bubble icon. The height must be provided as a string value in one of standard [length](https://developer.mozilla.org/en-US/docs/Web/CSS/length) units.

```js
Bots.setChatBubbleIconHeight('18px');
```

### setChatBubbleIconWidth(width)

Sets the width of loading chat bubble icon. The width must be provided as a string value in one of standard [length](https://developer.mozilla.org/en-US/docs/Web/CSS/length) units.

```js
Bots.setChatBubbleIconWidth('24px');
```

### setChatBubbleIconSize(width, height)

Sets the size of loading chat bubble icon. The width and height must be provided as string values in one of the standard [length](https://developer.mozilla.org/en-US/docs/Web/CSS/length) units.

```js
Bots.setChatBubbleIconSize('24px', '18px');
```

### setFont(font)

Sets the font for the chat widget. The syntax of `font` must follow the [standard pattern](https://developer.mozilla.org/en-US/docs/Web/CSS/font).

```js
Bots.setFont('14px italic large sans-serif');
```

### setFontSize(fontSize)

Sets the font size for the text in chat widget. The syntax of `fontSize` must follow the standard [permissible values](https://developer.mozilla.org/en-US/docs/Web/CSS/font-size).

```js
Bots.setFontSize('16px');
```

### setHeight(height)

Sets the height of chat widget. The value passed must be in one of [length](https://developer.mozilla.org/en-US/docs/Web/CSS/length) units.

```js
Bots.setHeight('786px')
```

### setWidth(width)

Sets the width of chat widget. The value passed must be in one of [length](https://developer.mozilla.org/en-US/docs/Web/CSS/length) units.

```js
Bots.setWidth('320px')
```

### setSize(width, height)

Sets the size of chat widget. The values passed must be in one of [length](https://developer.mozilla.org/en-US/docs/Web/CSS/length) units.

```js
Bots.setSize('320px' ,'786px',)
```

### setMessagePadding(padding)

Sets the padding around the message in the chat widget. The syntax of `padding` must follow standard [permissible values](https://developer.mozilla.org/en-US/docs/Web/CSS/padding).

```js
// Apply padding to all four sides
Bots.setMessagePadding('15px');
// vertical | horizontal
Bots.setMessagePadding('5% 10%');
// top | horizontal | bottom
Bots.setMessagePadding('15em 20em 20em');
// top | right | bottom | left
Bots.setMessagePadding('15px 20px 20px 15px');
```

### setTextColor(color)

Sets the primary text color of messages in chat widget. It does not modify the colors in actions. The value passed must be in standard [color](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value) format.

```js
Bots.setTextColor('#33ee00')
```

### setTextColorLight(color)

Sets the color of secondary messages in chat widget, like descriptions. The value passed must be in standard [color](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value) format.

```js
Bots.setTextColorLight('rgb(13, 155, 200)')
```


## Events

To bind an event, use `Bots.on(<event name>, <handler>)`. To unbind events, you can either:
* Call `Bots.off(<event name>, handler)` to remove one specific handler.
* Call `Bots.off(<event name>)` to remove all handlers for an event.
* Call `Bots.off()` to unbind all handlers.

It is preferable to add event handlers before calling `connect()` method, so that they can be registered and triggered appropriately.

### ready

This event is triggered when init completes successfully. Be sure to bind before calling init.

```js
Bots.on('ready', function(){
    console.log('the init has completed!');
});
```

Listeners to `ready` event must be added before calling the `connect()` method, otherwise the listeners will not be registered in time to be triggered.

### destroy

This event is triggered when the widget is destroyed.
```js
Bots.on('destroy', function(){
    console.log('the widget is destroyed!');
});
Bots.destroy();
```

### message:received

This event is triggered when the user receives a message.
```js
Bots.on('message:received', function(message) {
    console.log('the user received a message', message);
});
```

### message:sent

This event is triggered when the user sends a message.
```js
Bots.on('message:sent', function(message) {
    console.log('the user sent a message', message);
});
```

### message

This event is triggered when a message was added to the conversation.
```js
Bots.on('message', function(message) {
    console.log('a message was added to the conversation', message);
});
```

### networkstatuschange

This event is triggered whenever the network status of the connection to the server changes. The callback receives a status number parameter, which corresponds to [WebSocket network status constants](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket#Constants).
```js
Bots.on('networkstatuschange', function(status) {
    switch (status) {
        case 0:
            status = 'Connecting';
            break;
        case 1:
            status = 'Open';
            break;
        case 2:
            status = 'Closing';
            break;
        case 3:
            status = 'Closed';
            break;
    }
    console.log(status);
});
```

### unreadCount

This event is triggered when the number of unread messages changes.
```js
Bots.on('unreadCount', function(unreadCount) {
    console.log('the number of unread messages was updated', unreadCount);
});
```

### widget:opened

This event is triggered when the widget is opened.
```js
Bots.on('widget:opened', function() {
    console.log('Widget is opened!');
});
```

### widget:closed

This event is triggered when the widget is closed.
```js
Bots.on('widget:closed', function() {
    console.log('Widget is closed!');
});
```

## Features

### Autocomplete

`Feature Flag: enableAutocomplete`

`Enable client side caching: enableAutocompleteClientCache`

Autocomplete provides suggestions to end users to better form the utterance and select the specific entity value. For it to work, you must add autocomplete suggestions to the skill's intents. Then, whenever the user types a character in the input, the intents' autocomplete suggestions display in a popup to suggest the best format and to minimize user errors. An intent's autocomplete suggestions should not be a full word dictionary. Their purpose is to minimize the scope and allow better guidance to the end user.


### Delegation

`Feature configuration: delegate`

The delegation feature lets you set a delegate to receive callbacks before certain events in the conversation. To set a delegate, pass it in the `delegate` property or use the `setDelegate` method. The delegate object may optionally contain `beforeDisplay`, `beforeSend`, and `beforePostbackSend` delegate functions.

```js
const delegate = {
    beforeDisplay: function(message) {
        return message;
    },
    beforeSend: function(message) {
        return message;
    },
    beforePostbackSend: function(postback) {
        return postback;
    }
}

Bots.setDelegate(delegate);
```

#### beforeDisplay

The `beforeDisplay` delegate allows a bot message to be modified before it is displayed in the conversation. The message returned by the delegate is used to display the message. If it returns a falsy value, like `null`, `undefined`, or `false`, the message is not displayed.

#### beforeSend

The `beforeSend` delegate allows a user message to be modified before it is sent to the chat server. The message returned by the delegate is sent to bot. If it returns a falsy value, like `null`, `undefined`, or `false`, the message is not displayed.

#### beforePostbackSend

The `beforePostbackSend` delegate is similar to `beforeSend`, just applied to postback messages from user. The postback returned by the delegate is sent to bot. If it returns a falsy value, like `null`, `undefined`, or `false`, the message is not displayed.


### Embedded mode

`Feature flag: embedded`

`Pass ID of target container element: targetElement`

The chat widget appears as a fixed position element on the bottom right of the page, which does not respond to gestures on the page. But it can also be embedded into the host and become part of the page. To activate the embedded mode, you need to pass `embedded: true`, and provide a target container with `targetElement: <targetDivId>` . This accepts a DOM element which will be used as the container where the widget will be rendered.

The embedded widget occupies the full width and height of the container. A height and width must be specified for the target element. Otherwise, the widget will collapse.


### Headless SDK

`Feature flag: enableHeadless`

Similar to headless browsers, the SDK can also be used without a UI. The SDK maintains the connection to the server, and provides APIs to send messages, receive messages, and get updates of the to network status. Developers can use the APIs to interact with the SDK, and update UI appropriately.

The communication can be implemented as follows:
* Sending messages - Calls `Bots.sendMessage(message)` to pass any payload to server.
* Receiving messages - Response can be listened for using `Bots.on('message:received', <messageReceivedCallbackFunction>)`.
* Get connection status update - Updates to status of the connection to server can be listened for using `Bots.on('networkstatuschange', <networkStatusCallbackFunction>)`. The callback has a parameter `status`, which is updated with values from 0 to 3. These values map to [WebSocket states](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket#Constants):
  * 0 - WebSocket.CONNECTING - Connecting
  * 1 - WebSocket.OPEN - Open
  * 2 - WebSocket.CLOSING - Closing
  * 3 - WebSocket.CLOSED - Closed


### Long Polling

`Feature flag: enableLongPolling`

The SDK uses WebSockets to connect to the server and converse with skills. If for some reason the WebSocket is disabled over the network, traditional http calls can be used to chat with the bot. This feature is known as long polling, as the SDK has to continuously call or 'poll' the server to fetch the latest messages from the skill.

Long polling provides fallback support for conversations on browsers which don't have WebSockets enabled or blocked.


### Response narration

`Feature flag: enableBotAudioResponse`

`Functionality configuration: skillVoices`

The SDK can use the device's speech synthesis APIs to narrate the server's responses. You use the `skillVoices` array to set the voice preferences. Each item in the array must be an object that has a `lang` field and and an optional `name` field. The SDK looks up the availability of each voice on the device in the order that they are passed in the setting, and the first complete match is set as the voice. If no exact match is found, then the SDK uses the first match based on the `lang` value alone. If there's still no match, then the SDK uses the device's default language.

```js
const settings = {
    ...,
    enableBotAudioResponse: true,
    skillVoices: [{
        lang: 'en-US',
        name: 'Samantha'
    }, {
        lang: 'en-US',
        name: 'Alex'
    }, {
        lang: 'en-UK'
    }]
}
```


### Voice Recognition

`Feature flag: enableSpeech`

`Functionality configuration: enableSpeechAutoSend`

The SDK has been integrated with speech recognition to send messages in the form of voice to skills and digital assistants. It allows users to talk directly to the skill during conversations and get appropriate responses.

When the feature is enabled, a mic button appears in place of the send button whenever the user input field is empty. The user can click on the mic button to start voice recording and send it to the speech server for recognition. The speech is converted to text and sent to the skill/DA. If the speech is partially recognized, the partial result is set in the user input field and the user can clean it up before sending it.

The feature can also be utilized using two exposed methods - `startVoiceRecording(onSpeechRecognition, onSpeechNetworkChange)` to start recording, and `stopVoiceRecording()` to stop recording. The APIs are detailed in the Method section above.

The `enableSpeechAutoSend` flag setting lets you configure whether to send the text that's recognized from the user's voice directly to the chat server. When set to `true` (the default), the user's speech response is automatically sent to the chat server. When set to `false`, the user's speech response is rendered in the input text field before it is sent to the chat server so that users can manually modify it before sending, or delete the message.

## Message Model

To use features like headless mode and delegate, a clear understanding of bot and user messages is essential. Everything received from and sent to the chat server are represented as messages. This includes:
* Messages from user → bot (e.g. "I want to order a Pizza")
* Messages from bot → user (e.g. "What kind of crust do you want?")

The following section describes each of the message types in more detail.

### Base Types
These are the base types that are used by User → Bot messages and Bot → User messages. They are the building blocks for the messages.


#### Attachment
This represents an attachment sent User → Bot or Bot → User.

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| title | A title for the attachment | string | No |
| type | The type of attachment | string (valid values: audio, file, image, video) | Yes |
| URL | The URL to download the attachment | string | Yes |

Example JSON
```json
{
    "title": "Oracle Open World Promotion",
    "type": "image",
    "URL": "https://www.oracle.com/us/assets/hp07-oow17-promo-02-3737849.jpg"
}
```


#### Location
This represents a Location object.

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| title | A title for the location | string | No |
| URL | A URL for displaying a map of the location | string | No |
| latitude | The GPS coordinate's longitude value | double | Yes |
| longitude | The GPS coordinate's longitude value | double | Yes |

Example JSON
```json
{
    "title": "Oracle Headquarters",
    "URL": "https://www.google.com.au/maps/place/37°31'47.3%22N+122°15'57.6%22W",
    "longitude": -122.265987,
    "latitude": 37.529818
}
```


#### Action
An action represents something that the user can select. Every action includes the following properties:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | Action type for the action | string | Yes |
| label | Label to display for the action | string | At least one of `label` or `imageUrl` must be present |
| imageUrl | Image to display for the action | string | At least one of `label` or `imageUrl` must be present |


##### PostbackAction
This action will send a pre-defined postback back to the bot if the user selects the action.
It adds the following properties to the `Action` properties:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | Action type for the action | "postback" | Yes |
| postback | The postback to be sent back if the action is selected | string or JSONObject | Yes |

Example JSON
```json
{
    "type": "postback",
    "label": "Large Pizza",
    "imageUrl": "https://amicis.com/images/gallery/locations/11.jpg",
    "postback": {
        "state": "askSize",
        "action": "getCrust"
    }
}
```


##### CallAction
This action will request the client to call a specified phone number on the user's behalf.
It adds the following properties to the `Action` properties:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | Action type for the action | "call" | Yes |
| phoneNumber | The phone number to call | string | Yes |

Example JSON
```json
{
    "type": "call",
    "label": "Call Support",
    "imageUrl": "http://ahoraescuando.bluefm.com.ar/files/2016/05/cuidado.jpg",
    "phoneNumber": "18002231711"
}
```


##### UrlAction
This action will request the client to open a website in a new tab or in an in-app browser.
It adds the following properties to the `Action` properties:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | Action type for the action | "call" | Yes |
| URL | The URL of the website to display | string | Yes |


##### ShareAction
This action will request the client to open a sharing dialog for the user.
It adds the following properties to the `Action` properties:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | Action type for the action | "share" | Yes |


##### LocationAction
This action will request the client to ask the user for a location.
It adds the following properties to the `Action` properties:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | Action type for the action | "location" | Yes |

Example JSON
```json
{
    "type": "location",
    "label": "Share location",
    "imageUrl": "http://images.clipartpanda.com/location-clipart-location-pin-clipart-1.jpg"
}
```


#### Card
A card represents a single card in the message payload.  It contains the following properties:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| title | The title of the card, displayed as the first line on the card. | string | Yes |
| description | The description of the card | string | No |
| imageUrl | URL of the image that is displayed | string | No |
| URL | URL of a website that is opened when taping on the card | string | No |
| actions | An array of actions related to the text | Array<Action> | No |


### Conversation Messages
All messages that are part of the conversation are structured as follows:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| messagePayload | Message payload | Message | Yes |
| userId | User ID | string | Yes |

Example conversation message
```json
{
    "messagePayload": {
        "text": "show menu",
        "type": "text"
    },
    "userId": "guest"
}
```


### Message
Message is the abstract base type for all other messages. All messages extend it to provide more information.

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | string | Yes |


### User Message
This represents a Message sent from user → bot.


#### User Text Message
This is a simple text message sent to the server.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "text" | Yes |
| text | Text message | string | Yes |

```json
{
    "messagePayload": {
        "text": "Order Pizza",
        "type": "text"
    },
    "userId": "guest"
}
```


#### User Postback Message
This is the postback response message sent to the server.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "postback" | Yes |
| text | Text for postback | string | No |
| postback | The postback of the selected action | string or JSONObject | Yes |

```json
{
    "messagePayload": {
        "postback": {
            "variables": {
                "pizza": "Small"
            },
            "system.botId": "69F2D6BB-35BF-4BCA-99A0-A88D44A51B35",
            "system.state": "orderPizza"
        },
        "text": "Small",
        "type": "postback"
    },
    "userId": "guest"
}
```


#### User Attachment Message
This is the attachment response message sent to the server.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "attachment" | Yes |
| attachment | Attachment metadata | Attachment | Yes |

```json
{
    "messagePayload": {
        "attachment": {
            "type": "image",
            "URL": "http://oda-instance.com/attachment/v1/attachments/d43fd051-02cf-4c62-a422-313979eb9d55"
        },
        "type": "attachment"
    },
    "userId": "guest"
}
```


#### User Location Message
This is the location response message sent to the server.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "location" | Yes |
| location | User location information | Location | Yes |

```json
{
    "messagePayload": {
        "location": {
            "latitude": 45.9285271,
            "longitude": 132.6101925
        },
        "type": "location"
    },
    "userId": "guest"
}
```


### Bot Message
This represents a Message sent from Bot → User.


#### Bot Text Message
This represents a text message.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "text" | Yes |
| text | Text of the message | string | Yes |
| actions | An array of actions related to the text | Array<Action> | No |
| globalActions | An array of global actions related to the text | Array<Action> | No |

```json
{
    "messagePayload": {
        "type": "text",
        "text": "What do you want to do?",
        "actions": [
            {
                "type": "postback",
                "label": "Order Pizza",
                "postback": {
                    "state": "askAction",
                    "action": "orderPizza"
                }
            },
            {
                "type": "postback",
                "label": "Cancel A Previous Order",
                "postback": {
                    "state": "askAction",
                    "action": "cancelOrder"
                }
            }
        ]
    },
    "userId": "guest"
}
```


#### Bot Location Message
This represents a location message.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "location" | Yes |
| location | The location | Location | Yes |
| actions | An array of actions related to the text | Array<Action> | No |
| globalActions | An array of global actions related to the text | Array<Action> | No |


#### Bot Attachment Message
This represents an attachment message.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "attachment" | Yes |
| attachment | The attachment sent | Attachment | Yes |
| actions | An array of actions related to the text | Array<Action> | No |
| globalActions | An array of global actions related to the text | Array<Action> | No |


#### Bot Card Message
This represents a set of choices displayed to the user, either horizontally like a carousal or vertically like a list.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "card" | Yes |
| layout | Whether to display the cards horizontally or vertically | string (values: horizontal, vertical) | Yes |
| cards | Array of cards to be rendered | Array<Card> | Yes |
| headerText | A header text for cards | string | No |
| actions | An array of actions related to the text | Array<Action> | No |
| globalActions | An array of global actions related to the text | Array<Action> | No |

Example JSON
```json
{
    "messagePayload": {
        "type": "card",
        "layout": "horizontal",
        "cards": [
            {
                "title": "Hawaiian Pizza",
                "description": "Ham and pineapple on thin crust",
                "actions": [
                    {
                        "type": "postback",
                        "label": "Order Small",
                        "postback": {
                            "state": "GetOrder",
                            "variables": {
                                "pizzaType": "hawaiian",
                                "pizzaCrust": "thin",
                                "pizzaSize": "small"
                            }
                        }
                    },
                    {
                        "type": "postback",
                        "label": "Order Large",
                        "postback": {
                            "state": "GetOrder",
                            "variables": {
                                "pizzaType": "hawaiian",
                                "pizzaCrust": "thin",
                                "pizzaSize": "large"
                            }
                        }
                    }
                ]
            },
            {
                "title": "Cheese Pizza",
                "description": "Cheese pizza (i.e. pizza with NO toppings) on thick crust",
                "actions": [
                    {
                        "type": "postback",
                        "label": "Order Small",
                        "postback": {
                            "state": "GetOrder",
                            "variables": {
                                "pizzaType": "cheese",
                                "pizzaCrust": "thick",
                                "pizzaSize": "small"
                            }
                        }
                    },
                    {
                        "type": "postback",
                        "label": "Order Large",
                        "postback": {
                            "state": "GetOrder",
                            "variables": {
                                "pizzaType": "cheese",
                                "pizzaCrust": "thick",
                                "pizzaSize": "large"
                            }
                        }
                    }
                ]
            }
        ],
        "globalActions": [
            {
                "type": "call",
                "label": "Call for Help",
                "phoneNumber": "123456789"
            }
        ]
    },
    "userId": "guest"
}
```


#### Bot Postback Message
This represents a postback.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "postback" | Yes |
| text | Text of the message | string | No |
| postback | The postback | string or JSONObject | Yes |
| actions | An array of actions related to the text | Array<Action> | No |
| globalActions | An array of global actions related to the text | Array<Action> | No |


#### Bot Raw Message
This is used when a component creates the channel-specific payload itself.
It applies the following properties to the Message:

| Name | Description | Type | Required |
| ------ | ------ | ------ | ------ |
| type | The message type | "raw" | Yes |
| payload | The channel-specific payload | JSONObject | Yes |


## Limitations

There are a few limitations on the usage of current Web SDK.

* The responses from the skill may become incorrect if the network goes offline in between a chat and comes back online within a short time (such as within 10 seconds). This only happens when a server response is lost due to network failure. In such cases, the user may have to send the previous message again, or may need to start the conversation afresh. The skill can be designed to handle such cases.
* The attachment file size is limited to 25MB. Larger files can not be uploaded in this release.
* The SDK currently only supports a single connection to the ODA instance per userId per browser or device. If multiple connections are made using the same userId from several devices, or multiple browsers on single device, then only skill/DA responses will be synchronized on all clients, not user messages.
* On Cordova based hybrid Android apps build with SDK integration, the attachment sharing currently does not work.
