# Seamless-Posting Architecture Documentation

## TL;DR

This documentation is supposed to explain the architecture and the implementation detail of the seamless-posting feature of the Privly extension.

## What's Seamless-Posting

The new feature that enables users to directly publish Privly message in the host page safely and seamlessly, just as the reading process.

## What is the work flow of Seamless-Posting

In seamless-posting, we first create a new Privly link, then repeatly update it while user is inputting. If the user clicks cancel, the Privly link will be destroyed.

## Privly-Chrome and Privly-Application both contain codes related to seamless-posting, what are their roles?

### Privly-Application

Seamless-posting implements the following parts:

1. Seamless-posting template (called prototype view) and related JavaScript (called view adapter)
   
   > prototype view: `privly-application/templates/seamless.html.template`
   > view adapter: `privly-application/shared/javascripts/viewAdapters/seamless.js`
  
   This is the view layer. It will be loaded into an iframe (to ensure safety) and put inside the host page if user enables seamless-posting for an editable element. Users will write secure messages in this page, which is embedded in the host page.

   In default, the template provides a textarea which has green background color. When user is inputting in the textarea, view adapter will call network interfaces to update the link.
   
   This is named as prototype view, because it is a base template, which is supposed to be extended/inherited in the specific Privly Application.

2. Seamless-posting TTLSelect view and view adapter

   > prototype view: `privly-application/templates/seamless_ttlselect.html.template`
   > view adapter: `privly-application/shared/javascripts/viewAdapters/seamless_ttlselect.js`

   This will be loaded into an iframe and put inside the host page if user hovers on the Privly button after enabling seamless-posting. This view layer provides menu-style UI for user to change the seconds_until_burn(TTL) option.

3. Seamless-posting feature for Message and PlainPost application

   Message and PlainPost application has already implemented the seamless-posting (and seamless-posting TTLSelect) view.
   
   > Message App:
   > view: `privly-application/Message/seamless.html.subtemplate`
   > controller: `privly-application/Message/js/controller/seamless.js`
   > 
   > PlainPost App:
   > view: `privly-application/PlainPost/seamless.html.subtemplate`
   > controller: `privly-application/PlainPost/js/controller/seamless.js`

   See `privly-application/Message/js/controllers/seamless.js` for samples of hooking the creation process and deletion process of seamless-posting, to store and remove the encryption key of Message app in such processes.

   See `privly-application/Message/js/messageModel.js` for samples of manipulating created Privly link to append the encryption key, and providing own encrypted `structured_content`.

### Privly-Chrome

Seamless-posting implements the following parts:

1. Seamless-posting contentscript
   
   > Content script:
   > `javascripts/content_scripts/posting.*.js`
   > The content script is splitted into several files for better readibility.
   
   The content script mainly:
   - Creates a Privly button at the top-right corner when user focus on an editable element in the host page (`posting.button.js`).
   - Provides tooltip (`Clicks to enable Privly posting`) when user hovers on the button (`posting.tooltip.js`)
   - Creates the iframe of privly-application seamless-posting view (`app`) when user clicks the button (`posting.app.js`)
   - Inserts Privly link into the original editable element (`target`) after the view in iframe creates a Privly link (`posting.target.js`)
   - Provides a dropdown menu (`TTLSelect`) when user hovers on the button after enabling seamless-posting (`posting.ttlselect.js`)
   - Destroys the iframe if user clicks button again (`posting.app.js`)
   
   Notice that most of the feature above involve more than one content script file. See implementation section below for details.

2. Seamless-posting background script
   
   > Background script:
   > `javascripts/background_scripts/posting_process.js`
   > `javascripts/background_scripts/context_menu.js`
   > `javascripts/background_scripts/modal_button.js`
   
   The background script:
   - Creates a bridge for communicating between the content script and the Privly application (for example, which link to insert) (`posting_process.js`)
   - Pops up login dialog if the user is not logined (`posting_process.js`)
   - Creates context menu for user to select the desired App and enable seamless-posting (`context_menu.js`)
   - Update the icon of the action_button (`modal_button`) according to the status  of whether user is writing in a seamless-posting form (`modal_button.js`)

## Implementation of Privly-Application

> ECMAScript 6 Promise is heavily used to arrange the async callback order.

### Seamless-Posting View Adapter

#### Initialize process (`start`):

1. *(Privly-Chrome appends the iframe into the host page.)*

2. Send message to content script to switch the icon of Privly button to spinner icon (`msgStartLoading`).

3. Check connection.

  1. If fails: send message to content script to destroy the iframe (`msgAppClosed`) and restore the Privly button icon to lock icon and stops (`msgStopLoading`) and pops up login dialog (`msgPopupLoginDialog`).

  2. If succeeds: goto 4.

4. Send message to content script to retrive the original content of the editable element (`target`) (`msgGetTargetText`).

5. Check whether the original content contains a Privly link.

  1. If contains: try to load it (`loadLink`).

    1. If loaded succeeded and the link is a valid Privly link and the user has edit permission: Use the content of the Privly link as the initial content of the posting form (`initial content = (content of target).replace(the privly link, the content of the privly link)`), Goto 9.

    2. Else: Goto 6.

  2. If not contains: Goto 6.

6. Create a new and empty Privly link (`createLink`) and use it as the main link.

7. Send message to content script to insert the link into the editable element (`target`) (`msgInsertLink`).

8. Initialize using a new link completes. Goto 10.

9. Initialize using an existing link completes. Goto 10.

10. Set up targetContentMonitor (`beginContentClearObserver`), monitoring whether the content of the target contains our main link, once cleared, send message to content script to destroy this iframe (`msgAppClosed`).

11. Send message to content script switch the icon of Privly button from spinner icon to original icon (`msgStopLoading`).

12. Send message to content script to notify the completion of initial process (`msgAppStarted`).


#### Destroy process:

1. *(Privly-Chrome trying to destroy the iframe)*

2. *(Privly-Chrome send message to the iframe that it is going to be destroyed)*

3. Destroy the main link (`deleteLink`).

4. Send message to content script to notify the closing (`msgAppClosed`).

5. *(Privly-Chrome removes iframe from DOM tree)*

#### Create link (`createLink`):

1. Call `privlyNetworkService.sameOriginPostRequest` to create a link.

2. Call Application Model to post-process the link (`application.postprocessLink`).

3. Emit `afterCreateLink` event for Application Models.

4. Use the link as the main link.

#### Update link (`updateLink`):

1. Cancel last ongoing update request.

2. Call Application Model to transform textarea content into structured content (`application.getRequestContent`).

3. Call `privlyNetworkService.sameOriginPutRequest` to update the main link according to the structured content.

4. Emit `afterUpdateLink` event for Application Models.

#### Delete link （`deleteLink`):

1. Send message to content script to switch the icon of Privly button to spinner icon (`msgStartLoading`).

2. Call `privlyNetworkService.sameOriginDeleteRequest` to delete the main link

3. Emit `afterDeleteLink` event for Application Models.

4. Send message to content script to switch the icon of Privly button to original icon (`msgStopLoading`).

#### When user input something (`onHitKey`):

1. Update link (`updateLink`)

2. If it is keypress enter: send message to content script to simulate keypress enter event on the editable element (`target`) (`msgEmitEnterEvent`)

> PS: Sending message to content script is achived by sending message to background script and background script forwarding message to the content script.

### Seamless-Posting TTLSelect View Adapter

#### Initialize process (`start`):

1. Get TTL options from Application Model (`application.getTTLOptions`)

2. Calculate width and height

3. Send message to content script to notify the completion of loading, containing the width and height

4. *(Privly-Chrome resize the iframe and calculates the position of the iframe, based on the width and height. The iframe may be below the button, or above the button)*

5. *(Privly-Chrome send message to iframe to notify whether it is below the button or above the button)*

6. Generate menu DOM according to position: smaller options are always closer to the mousr-pointer.

7. *(Privly-Chrome fade in the iframe)*

#### When user clicks something (`onItemSelected`):

1. Send message to content script to notify that user has clicked an option (`msgTTLChange`)

> PS: Sending message to content script is achived by sending message to background script and background script forwarding message to the content script.

## Implementation of Privly-Chrome

### Content Script

#### Resource, ResourceItem

We have to manage state for editable elements on the page (for example, whether it is in seamless-posting mode). We also need to manage state for programmly-created elements (for example, a Privly button may be spinner icon, or lock button, or cancel button, based on state). Besides, we also need to manage some global objects (for example, a Privly button may contain a timer to postpone the hiding process). In addition, those stuff should be treated together: when editable element is removed, our state data should be cleared, our Privly button DOM related to that editable element should be removed and our Privly button timer should be canceled.

Thus we created the `Resource` class (implemented in `posting.resource.js`), to provide a container for those components (`ResourceItem`, implemented in `posting.resource.js`), for a specific editable element.

Features:

- Different editable element is linked to different `Resource`s.

- A `Resource` can hold many different `ResourceItem`s.

- A Privly button DOM is managed by a `ResourceItem` (implemented in `posting.button.js`), an editable element DOM itself is managed by a `ResourceItem` (implemented in `posting.target.js`), a Privly button tooltip DOM is managed by a `ResourceItem` (implemented in `posting.tooltip.js`), a Privly seamless-posting Application iframe DOM is managed by a `ResourceItem` (implemented in `posting.app.js`), etc.

- There is a global pool of `Resource` (managing all `Resource` instance).

- `ResourceItem` has `destroy` method, which is a kind of destructor.

- Different `ResourceItem` can have different destructor behaviours, for example, for a Privly button Resource Item, when it is destroyed, the DOM should be removed from DOM tree. However for a Target Resource Item, when it is destroyed, the DOM (the editable element itself) should not be removed from DOM tree. Notice that, Privly button Resource Item also cancels its timers inside `destroy`.

- Some `ResourceItem` may not contain DOM nodes, for example, `posting.controller.js` implements a `ResourceItem` which only control other `ResourceItem`s.

- `ResourceItem`s inside the same `Resource` can communicate with others by calling `broadcastInternal` (It is the observer design pattern).

- Each `Resource` has a unique id, theoretically (even across all tabs).

- When a Privly Application want to communicate with a specific `Resource`, it need to provide the id of the `Resource`.

- The id of the `Resource` is passed by URI querystring to the privly-application when iframe is created, thus the application can send message back later.

- When `Resource` receives a message, it will forward it to its `ResourceItem`s.

- A message received by `Resource` can be external -- Chrome message from background script,  or internal -- by calling `broadcastInternal()`.

- Messages send from `broadcastInternal` does not support `sendResponse`.

- A `ResourceItem` can subscribe different kind of message (differentiate by `action` property) by calling `addMessageListener`.

- We appoint that, messages of which `action` start with `posting/internal/` are internal messages (which means, when you are using `broadcastInternal`, the `action` property of your messages should be `posting/internal/` prefixed).

- The background script will forward Chrome messages of which `action` start with `posting/app` to all Privly applications (each Privly applications will filter messages).

- The background script will forward Chrome messages of which `action` start with `posting/contentScript` to all content scripts (each `Resource` will filter messages as metioned above).

- The background script itself will try to handle Chrome messages of which `action` start with `posting/background`.

#### Resource State

A `Resource` have a `state` property, indicates whether it is in the seamless-posting mode.

`state == OPEN`: In seamless-posting mode (posting form is open).

`state == CLOSE`: Not in seamless-posting mode (posting form is closed).

#### Button Internal State

Button has three kind of icons, `lock`, `spinner` and `cancel`.

Button also has two properties indicates its state:

`loading`: whether the button should show a spinner icon.

`state`: the same to Resource State.

We saparated the two properties thus each `ResourceItem` can set one of them without caring about affecting real states.

- For `loading == true`: internal state is `LOADING`, The button will show `spinner` icon.

- For `loading == false` and `state == CLOSE`: internal state is `CLOSE`, the button will show `lock` button.

- For `loading == false` and `state == OPEN`: internal state is `OPEN`, the button will show `cancel` button.

`INTERNAL_STATE_PROPERTY` defines behaviours of the button in each internal state:

```js
var INTERNAL_STATE_PROPERTY = {
  CLOSE: {
    autohide: true,  // whether the button should hide after seconds
    clickable: true, // whether the button is clickable
    tooltip: true,   // whether to show the tooltip when hovering on the button
    icon: SVG_OPEN   // the SVG of the icon when button is in this state
  },
  OPEN: {
    autohide: false,
    clickable: true,
    tooltip: false,
    icon: SVG_CLOSE
  },
  LOADING: {
    autohide: false,
    clickable: false,
    tooltip: false,
    icon: SVG_LOADING
  }
};
```

#### ContextId, ResId, AppId

Each iframe in each tab contains a unique `contextid`, which is generated in `context_messenger.js`, to filter contexts for messages (notice that every Chrome message is broadcasted to all tabs, so we filter the message at the context layer first).

Each `Resource` contains a unique `resid`, which is used to tell Privly applications that where to send Chrome message back later. It is generated by `Resource` class when constructing. Each `Resource` filter messages according to the `resourceid` property in the message body to ensure that the message is indeed send to this `Resource` and then broadcast them to its `ResourceItem`s.

Each Seamless-posting App contains a `appid`, which is used for Privly applications to filter messages sent from content script. It is generated by ResourceItem that creates the iframe (`posting.app.js` or `posting.ttlselect.js`). Again, it is used to ensure that such message is indeed send to this Privly application.

---

The process of showing the Privly button:

#### `posting.service.js`

1. `posting.service.js` adds `click`, `focus`, `blur` event listener to the document of the host page.

2. *(User clicks an editable element)*

3. `posting.service.js` detects whether it is an editable element and calculates the correct target element

4. If there are no `Resource` containing the target element, create one (`createResource@posting.service.js`):

  1. Create ControllerResourceItem implemented in `posting.controller.js`

  2. Create TargetResourceItem implemented in `posting.target.js`

  3. Create ButtonResourceItem implemented in `posting.button.js`

  4. Create TooltipResourceItem implemented in `posting.tooltip.js`

  5. Create TTLSelectResourceItem implemented in `posting.ttlselect.js`

  6. Create a `Resource` containing `ResourceItem`s above and add it to the global `Resource` pool.

5. Send internal message to all `ResourceItem` of the `Resource` that the target element is activated (`posting/internal/targetActivated`)

#### When `posting.controller.js` constructs `ControllerResourceItem`

1. `// Nothing other than add message listeners`

#### When `posting.target.js` constructs `TargetResourceItem`

1. Store the target node in the `ResourceItem`

#### When `posting.button.js` constructs `ButtonResourceItem`

1. Create the DOM node for the button

2. Add event listeners

3. Set the button icon to lock (`updateInternalState`)

#### When `posting.tooltip.js` constructs `TooltipResourceItem`

1. Create DOM for the tooltip (see `posting.floating.js` for underlayer implementation)

#### When `posting.ttlselect.js` constructs `TTLSelectResourceItem`

1. Create DOM for the TTLSelect (see `posting.floating.js` for underlayer implementation)

#### When `posting.target.js` receives `internal/targetActivated`

1. `posting.target.js` starts the resize monitor (`updateResizeMonitor`) to detect whether the position or size of the target element has changed (`detectResize`).

2. If position or size changed, send internal message to all `ResourceItem` of the `Resource`: `posting/internal/targetPositionChanged`

#### When `posting.target.js` receives `internal/targetDeactivated`

1. `posting.target.js` stops the resize monitor (`updateResizeMonitor`) if this `Resource` is not in `OPEN` state.

#### When `posting.button.js` receives `internal/targetActivated`

1. Update the position of the button (`updatePosition`) according to the position of the target

2. Show the button

3. Set a timer to postpone hiding

#### When `posting.button.js` receives `internal/targetDeactivated`

1. Cancel the postpone timer.

2. Immediately hide the button if it should be hidden according to internal state and `INTERNAL_STATE_PROPERTY`.

#### When `posting.button.js` receives `internal/targetPositionChanged`

1. Updates the position of the button

---

The process of starting seamless-posting or stoping seamless-posting:

#### When `posting.button.js` receives onClick event of the Privly Button DOM

1. Stop if the button is not in a clickable state

2. Send internal message to all `ResourceItem` of the `Resource`: `posting/internal/buttonClicked`

#### When `posting.controller.js` receives `internal/buttonClicked`

1. If the `Resource` is in `CLOSE` state (seamless-posting is not enabled and user clicks the Privly button thus the user is going to enable seamless-posting): Create `AppResourceItem` implemented in `posting.app.js`

2. If the `Resource` is in `OPEN` state: Send internal message to all `ResourceItem` of the `Resource`: `posting/internal/closeRequested` (user has requested to close seamless-posting form)

#### When `posting.app.js` constructs `AppResourceItem`

1. Generate a unique app id

2. Creates an iframe, which `src` is like `privly-applications/Message/seamless.html?contextid=xxx&resid=xxx&appid=xxx`.

3. Append to DOM tree

#### When `posting.app.js` receives `internal/closeRequested`

1. Send message to the app: `posting/app/userClose`

---

The process of hovering on the Privly button:

#### When `posting.button.js` receives onMouseEnter event of the Privly Button DOM

1. Send internal message to all `ResourceItem` of the `Resource`: `posting/internal/buttonMouseEntered`

#### When `posting.controller.js` receives `internal/buttonMouseEntered`

1. Call `tooltip.show` if the `Resource` is in `CLOSE` state

2. Call `ttltooltip.show` if the `Resource` is in `OPEN` state

> There are many other event or message handling process, please check out the code to see others. Event or message handlers above are those typical ones, which I think is enough to give you a comprehensive understanding of the content script architecture.
