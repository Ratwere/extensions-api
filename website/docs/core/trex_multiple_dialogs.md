---
title: Add Multiple Popup Dialog Boxes
description: How to open multiple popup dialog boxes from an extension and send messages between them
---

```mdx-code-block
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import CodeBlock from '@theme/CodeBlock';
import { Link } from "react-router-dom";
```

Starting with Tableau 2026.1, the Extensions API supports multiple popup windows or dialog boxes, and supports messaging between them. In addition to the optional configuration popup that you can create when you initialize the extension, you can open other dialog boxes, by adding buttons and controls to your extension.

If you're creating multiple dialog boxes, you must consider how you want them to interact with each other and the user. The popup windows can be modal or modeless, and they can share information and payloads. The Extensions API provides two new methods for sending text messages to and from the extension to the popup windows, and from the popup windows to the extension or calling popup window. If you have multiple popup windows, you must track which popup window is active, so you can control their interactions.

By design, there can be only one context configuration popup window or dialog box. This context menu item is explicitly declared in the `.trex` file of the extension. For additional popup windows, you don't need to modify the `.trex` file. For information about creating and using the configuration window, see [Add a Configuration Popup Dialog Box](./trex_configure.md).

## Create additional popup windows

You can create more than one popup window to provide configuration options for your extension. If you’re creating more popup windows, you must call those popup windows from methods other than the `configure()` method, and you must add controls to call those methods.

You can't display these other dialog boxes or windows directly when the user selects the configure context item. However, you can display the other popup windows when users click buttons and controls in your extension, or when users click controls in your configuration dialog box. 

### Basic steps

* Determine how you want the popup windows to interact (if at all). Should they be modal or modeless? A modal dialog box takes the focus and blocks all other windows until the dialog box is closed. A modeless dialog box does not block other windows, and the focus switches depending upon the user's selections.

* Do the popup windows need to share information or content with the extension? The Extensions API provides methods for sending text (string) messages between the popup windows and the extension.

* To make it easy to manage your popup windows, create separate HTML and JavaScript files for each window.

* Determine where to place the controls to display the popup windows. For example, in your main extension window, or in the configuration dialog. Add the controls (buttons, toolbars) to your extension HTML code to create the additional dialog boxes or popup windows.

* Use the [`displayDialogAsync()`](pathname:///api/interfaces/ui.html#displaydialogasync) method to display the popup window. You would probably call the `displayDialogAsync()` method from a wrapper that gets triggered when the user clicks a button or icon in a toolbar. In your extension JavaScript or TypeScript code, define your popup windows: the `popupURL`, the dialog style, and dimensions, and any initial payload you want to send to the popup window. And provide any necessary code for handling the payload that's returned from the popup window. 

  Like the configuration popup, the URL of the additional popup windows must belong to the same domain as the parent extension. The relative paths must resolve to the directory, or a child directory, of the extension. Root-relative paths aren’t allowed. For example, `./config.html` or `config.html` are allowed, but not the root-relative path `/config.html`.

* In your HTML and JavaScript code for your popup window, call the [`initializeDialogAsync()`](pathname:///api/interfaces/extensions.html#initializedialogasync) method and add code to process the initial payload, if you’re using it. In addition to whatever features you want the popup window to do, add a control (for example, a Save button) that you can attach to the `closeDialog()` method. The `closeDialog()` method safely closes the dialog box and returns the payload to the calling extension (or window).

* If you have more than one popup window, you must create a system for tracking the state of each popup window. For example, calling `displayDialogAsync()` method for an already-open dialog box throws an error. And if you only want one popup window open at a time, and you don't want them to be modal, you must track which popup is open and which popup is queued up. A simple Map or Flag variable can accomplish this tracking.

### Example: Add color selector dialog box

For example, you could add another dialog box to the [uiNamespace](https://github.com/tableau/extensions-api/tree/main/Samples/Dashboard/UINamespace) sample that lets users set the background color of the extension. To do this, you could add a button to display the popup window in the `uiNamespace.html` file.  When the user clicks the button, a color formatting dialog opens. The user can select the background color. When the user clicks a Save button, it calls the `closeDialog()` method and closes the popup window, which returns the new background color as the payload, and the new background color is applied.

:::tip

For the complete listing of the uiNamespace example that shows the addition of a second popup dialog box, see  
[Complete example source code](#complete-example-source-code).

:::

HTML code (uiNamespace.html)

```html
<!--  button declared in uiNamespace.html -->
<button id="openColorDialog" class="btn btn-secondary" style="margin-top: 10px; margin-left: 6px">Set Background Color</button>

```

JavaScript code (uiNamespace.js)

```javascript

// in uiNamespace.js, a button click calls openColorDialog

    $('#openColorDialog').click(openColorDialog);

```

In the main extension file, `uiNamespace.js`, add code to set the default background color and then call the `displayDialogAsync()` method. To see the entire listing, see [Complete example source code](#complete-example-source-code) and select the **uiNamespace** tab.  

```javascript

/**
* uiNamespace.js (main extension file)
* Sets the default background color
* Passes the current color to the dialog as the initial payload
*/

const defaultBgColor = '#FFFFFF';
let currentBgColor = defaultBgColor;


```

The `openColorDialog()` method calls `tableau.extensions.ui.displayDialogAsync()` method, and passes values for the popup window, including the URL, the initial payload, dimensions, and the `DialogStyle`.  The `DialogStyle` determines how the popup window interacts with the extension and other elements.

| `DialogStyle` value | Behavior |
|---|---|
| `tableau.DialogStyle.Modal` | Blocks all other windows; users must close this dialog before interacting with the extension. |
| `tableau.DialogStyle.Modeless` | Doesn't block other windows; focus follows user interaction. |
| `tableau.DialogStyle.Window` | Opens as a separate browser-style window; behavior depends on the browser. |

In this code snippet, the dialog style is set to modeless. The complete example extends this to allow the user to choose the dialog style. See [Complete example source code](#complete-example-source-code) and select the **uiNamespace** tab

```javascript

/**  uiNamespace.js 
* Defines and displays the dialog box
* Opens the background color picker dialog ( in this case, modeless).
* Passes the current background color as the initial payload so the dialog
* can pre-select it.  The closePayload returned is the chosen hex color,
* which is then applied to the extension body.
* 
*/

function openColorDialog () {
  const popupUrl = new URL('uiNamespaceColorDialog.html', window.location.href).href;

  let dialogStyle = tableau.DialogStyle.Modeless;

  tableau.extensions.ui
    .displayDialogAsync(popupUrl, currentBgColor, { height: 500, width: 360, dialogStyle: dialogStyle})
    .then((closePayload) => {
      currentBgColor = closePayload;
      $('body').css('background-color', currentBgColor);
    })
    .catch((error) => {
      if (error.errorCode === tableau.ErrorCodes.DialogClosedByUser) {
        console.log('Color dialog was closed by user');
      } else {
        console.error(error.message);
      }
    });
}

```

JavaScript code (uiNameSpaceColorDialog.js)

In the `uiNameSpaceColorDialog.js` file, add the `initializeDialogAsync()` method. The method processes the initial payload, in this case the current background color. If the user selects a color, that color is returned as the payload in the `closeDialog()` method. The `buildColorList()` and `updatePreview()` methods aren't shown here, but you can see the [Complete example source code](#complete-example-source-code). Those methods let users pick from the colors in a list and then see that color in a preview `div`.

When the user clicks the **Save** button, the `sendToParent()` method is called. The `sendToParent()` method is defined in the `uiNameSpaceColorDialog.js` file, and is described under [Send the message to the parent extension or popup window](#send-the-message-to-the-parent-extension-or-popup-window). For information about messaging between windows, see [Send messages to and from the popup window or dialog box](#send-messages-to-and-from-the-popup-window-or-dialog-box).

```javascript

// uiNameSpaceColorDialog.js

$(document).ready(function () {
  tableau.extensions.initializeDialogAsync().then(function (openPayload) {
    // openPayload is the current background color passed from the parent.
    if (openPayload && openPayload.startsWith('#')) {
      selectedColor = openPayload;
    }

    buildColorList();
    updatePreview(selectedColor);

    $('#saveButton').click(function () {
      sendToParent('Hello from Dialog box! - Save button clicked');
      tableau.extensions.ui.closeDialog(selectedColor);
    });
  });
});


```

---

## Send messages between popup dialog boxes and the extension

Starting with Tableau 2026.1, the Extensions API provides two methods for sending messages between dialog boxes and between the dialog box and the calling extension or popup window. The messages provide a way of passing information between the popup windows. The messaging is in addition to the optional payload that is returned from the `displayDialogAsync()` method. When the popup window calls the `closeDialog()` method, the `displayDialogAsync()` method returns the payload. The `sendDialogMessageAsync()` and `sendDialogMessageToParentAsync()` methods provide another way to communicate between popup windows and the main extension window.

### Add the message received event listener

To enable messaging between the popup windows, you must add an event listener for the `DialogMessageReceived` event. For two-way communication between popup windows and the extension, add the event listener in the code for both the popup window and the extension. 

For example, the following code snippet shows how a popup window could add the event listener for messages sent from the extension. These are the messages that the extension sends using the `sendDialogMessageAsync()` method. This example logs the message to the console window, and is viewable if you use the debugging tools on the popup window. 

Popup window or dialog box code:

```javascript

  tableau.extensions.ui.addEventListener(tableau.TableauEventType.DialogMessageReceived, (event) =>{
      console.log(`Message received: ${event.message}`);
  });

```

Extension code:

In the extension, add an event listener for messages that get sent from the popup window. In this case, the code snippet appends the message from the popup window to the `messageLog` text block in the extension interface.

```javascript
  // Listen for messages from dialogs sent by sendDialogMessageToParentAsync)
  tableau.extensions.ui.addEventListener(tableau.TableauEventType.DialogMessageReceived, function (event) {
    const p = $('<p />').text(event.message);
    $('#messageLog').append(p);
  });

```

### Send the message to the parent extension or popup window

The popup windows can use the `sendDialogMessageToParentAsync()` method to send a message to the extension or the popup window that opened it (that is, the "parent" of the popup dialog).

The `sendDialogMessageToParentAsync()` method takes just one argument, the message to send (a string value). The method only works if it’s called from the popup window.

For example, the following code snippets show how you might tie the send message method to a button in the popup window. In this case, there’s a **Save** button (`saveButton`). When the user clicks the **Save** button, a message gets sent to the extension. The `closeDialog()` method is also called when the user clicks the **Save** button, which closes the popup window and then returns the payload to the extension. We wrap the `sendDialogMessageToParentAsync()` method in another function with the `await` keyword, so that the message is guaranteed to be sent before the dialog box is closed.  

The parent extension's event handler function for the `DialogMessageReceived` event processes the message. 

```javascript

  // ...
  
  $('#saveButton').click(function () {
    sendToParent('Hello from Dialog box! - Save button clicked');
    tableau.extensions.ui.closeDialog(selectedColor);
  });

  // ...
  // send message method is placed in a wrapper function to guarantee 
  // that the message is sent before the dialog is closed
  async function sendToParent(message) {
    await tableau.extensions.ui.sendDialogMessageToParentAsync(message);
  }

```

### Send messages to and from the popup window or dialog box

The extension can use the `sendDialogMessageAsync()` method to send a message to the popup window or dialog box. The method takes two arguments, the message (string) and the optional target URL (`targetDialogUrl`).

The `sendDialogMessageAsync()` method can also be used to send messages to the extension from the popup window or dialog box. You might do this if the dialog box was opened by another dialog box (the "parent"), and not by the extension. However, in most cases, to send a message to the extension or to the dialog box that opened the dialog box sending the message (the "parent"), use the `sendDialogMessageToParentAsync()` method. 

If you are using the `sendDialogMessageAsync()` method to send a message to the extension from the popup window or dialog box, you can omit the target URL. When the target URL isn't specified, the message gets sent to the extension.

For example, in the following code snippet, a button in the extension code is tied to a `sendToColorDialog` method.  To be able to click the button to send a message, the colorDialog can't have the focus, so the popup dialog box must be modeless.

```javascript


  // Assumes the extension HTML code specifies a button 
  // with the id 'sendToColorDialog'
  $('#sendToColorDialog').click(sendToColorDialog);

  // ...

  // sends message to the popup window
  function sendToColorDialog() {
    let colorDialogUrl = new URL('uiNamespaceColorDialog.html', window.location.href).href; 
      tableau.extensions.ui.sendDialogMessageAsync('Hello from extension!', colorDialogUrl);
  }

```

---

## Track the active popup dialog box

Depending upon your implementation, and whether your dialog boxes are modal or not, if you have multiple popup dialog boxes you will likely need to track which popup is active. At a minimum, you'll want to prevent errors that result if a user tries to open a dialog box that is already open. This can be done with a simple mapping variable. In this case, we declare a set of objects called `openDialog`. The popup dialog boxes can then be added or deleted from the list, depending upon whether they are opened or closed.

```javascript

  // Track which dialogs are currently open
  const openDialogs = new Set();

```

### Add the opened popup window to the list

In your JavaScript code that calls the `displayDialogAsync()` method, check if the popup dialog box is currently open. If it is, return without calling the `displayDialogAsync()` method. If it is not open, add the URL to the list and continue. 

```javascript

  // code excerpt from openColorDialog() method

    const popupUrl = new URL('uiNamespaceColorDialog.html', window.location.href).href;

    if (openDialogs.has(popupUrl)) {
      return; // Dialog is already open; don't open a second instance
    }

    openDialogs.add(popupUrl);

```

### Remove the closed popup window from the list

In your function that handles the promise returned from the `displayDialogAsync()` method, remove the popup URL from the set of open dialog boxes, using the `delete()` method (`openDialogs.delete(popupUrl)`). Be sure to delete the popup dialog method in your error handling code. In the case of errors, where the dialog box isn't closed properly, for example, the Extensions API `closeDialog()` method isn't called, the dialog box is no longer open and needs to be removed from the list.

```javascript

  tableau.extensions.ui
    .displayDialogAsync(popupUrl, currentBgColor, { height: 500, width: 360, dialogStyle: dialogStyle })
    .then((closePayload) => {
      openDialogs.delete(popupUrl);
      currentBgColor = closePayload;
      $('body').css('background-color', currentBgColor);
    })
    .catch((error) => {
      openDialogs.delete(popupUrl);
      if (error.errorCode === tableau.ErrorCodes.DialogClosedByUser) {
        console.log('Color dialog was closed by user');
      } else {
        console.error(error.message);
      }
    });


```


---

## Complete example source code

This example code adds a second dialog box to the uiNamespace sample. The second popup dialog lets users change the background color of the extension window. The sample also shows how you can use the messaging methods to communicate between the popup dialog boxes and the extension. A simple mapping object is used to track the state of the second popup dialog box. 

Click the name of the module to see the HTML and JavaScript code.

---

```mdx-code-block
<Tabs queryString="Complete example code">
<TabItem value="uiN" label="uiNamespace" default>

```

* [HTML](#uinamespacehtml)
* [JavaScript](#uinamespacejs)
* [.trex](#uinamespacetrex)

#### uiNamespace.html

```html

<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Settings Sample</title>

    <!-- jQuery -->
    <script src="https://code.jquery.com/jquery-3.2.1.min.js"></script>

    <!-- Bootstrap -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css" >
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js" ></script>

    <!-- Extensions Library (this will be hosted on a CDN eventually) -->
    <script src="tableau.extensions.1.latest.js"></script>

    <!-- Our extension's code -->
    <script src="uiNamespace.js"></script>
  </head>
  <body>
    <div class="container">
      <p id="mainContent">
        <div id="active" style="display: none;">
          Refreshing <b><span id="datasourceCount">0</span></b> datasource(s) every <b><span id="interval">5</span></b> minutes.
        </div>
        <div id="inactive">
          Configure extension to proceed.
        </div>
      </p>
      <div class="container" style="border:1px solid #cecece">
        <h6>Dialog Style</h6>
        <div class="form-check">
          <input class="form-check-input" type="radio" name="dialogStyleRadio" id="modalStyle" checked>
          <label class="form-check-label" for="modalStyle">
            Modal
          </label>
        </div>
        <div class="form-check">
          <input class="form-check-input" type="radio" name="dialogStyleRadio" id="modelessStyle">
          <label class="form-check-label" for="modelessStyle">
            Modeless
          </label>
        </div>
        <div class="form-check">
          <input class="form-check-input" type="radio" name="dialogStyleRadio" id="windowStyle">
          <label class="form-check-label" for="windowStyle">
            Window
          </label>
        </div>
        <button id="openColorDialog" class="btn btn-secondary" style="margin-top: 10px; margin-left: 6px">Set Background Color</button>
        <button id="sendToColorDialog" class="btn btn-secondary" style="margin-top: 10px; margin-left: 6px">Send message to Color Dialog</button>
      </div>
      <div class="container" style="border:1px solid #cecece; margin-top: 10px">
        <h6>Messages from dialogs</h6>
        <div id="messageLog" style="max-height: 120px; overflow-y: auto; font-size: 0.9em;"></div>
      </div>
    </div>
  </body>
</html>



```

#### uiNamespace.js

```javascript


'use strict';

/**
 * UINamespace Sample Extension
 *
 * This sample extension demonstrates how to use the UI namespace
 * to create a popup dialog with additional UI that the user can interact with.
 * The content in this dialog is actually an extension as well (see the
 * uiNamespaceDialog.js for details).
 *
 * This sample is an extension that auto refreshes datasources in the background of
 * a dashboard.  The extension has little need to take up much dashboard space, except
 * when the user needs to adjust settings, so the UI namespace is used for that.
 */

// Wrap everything in an anonymous function to avoid polluting the global namespace
(function () {
  const defaultIntervalInMin = '5';
  const defaultBgColor = '#FFFFFF';
  let activeDatasourceIdList = [];
  let currentBgColor = defaultBgColor;

  // Track which dialogs are currently open
  const openDialogs = new Set();

  $(document).ready(function () {
    // When initializing an extension, an optional object is passed that maps a special ID (which
    // must be 'configure') to a function.  This, in conjunction with adding the correct context menu
    // item to the manifest, will add a new "Configure..." context menu item to the zone of extension
    // inside a dashboard.  When that context menu item is clicked by the user, the function passed
    // here will be executed.
    tableau.extensions.initializeAsync({ configure: configure }).then(function () {
      // This event allows for the parent extension and popup extension to keep their
      // settings in sync.  This event will be triggered any time a setting is
      // changed for this extension, in the parent or popup (i.e. when settings.saveAsync is called).
      tableau.extensions.settings.addEventListener(tableau.TableauEventType.SettingsChanged, (settingsEvent) => {
        updateExtensionBasedOnSettings(settingsEvent.newSettings);
      });

      // Listen for messages from dialogs (for example, the background color dialog box via sendDialogMessageToParentAsync)
      tableau.extensions.ui.addEventListener(tableau.TableauEventType.DialogMessageReceived, function (event) {
        const p = $('<p />').text(event.message);
        $('#messageLog').append(p);
      });

      $('#openColorDialog').click(openColorDialog);
      $('#sendToColorDialog').click(sendToColorDialog);
      $('#sendToColorDialog').prop('disabled', true);
    });
  });

  function configure () {
    // This uses the window.location.origin property to retrieve the scheme, hostname, and
    // port where the parent extension is currently running, so this string doesn't have
    // to be updated if the extension is deployed to a new location.
    const popupUrl = `${window.location.origin}/uiNamespaceDialog.html`;


    // This checks for the selected dialog style in the radio form.
    let dialogStyle;
    const dialogStyleOptions = document.getElementsByName('dialogStyleRadio');
    if (dialogStyleOptions[0].checked) {
      dialogStyle = tableau.DialogStyle.Modal;
    } else if (dialogStyleOptions[1].checked) {
      dialogStyle = tableau.DialogStyle.Modeless;
    } else {
      dialogStyle = tableau.DialogStyle.Window;
    }

    /**
     * This is the API call that actually displays the popup extension to the user.  
     * The only required parameter is the URL of the popup,
     * which must be the same domain, port, and scheme as the parent extension. 
     * The dialog style (modal, modeless, or window) is selected by the user.
     *
     * The developer can optionally control the initial size of the extension by passing in
     * an object with height and width properties.  The developer can also pass a string as the
     * 'initial' payload to the popup extension.  This payload is made available immediately to
     * the popup extension.  In this example, the value '5' is passed, which will serve as the
     * default interval of refresh.
     */
    tableau.extensions.ui
      .displayDialogAsync(popupUrl, defaultIntervalInMin, { height: 500, width: 500, dialogStyle })
      .then((closePayload) => {
        // The promise is resolved when the dialog has been expectedly closed, meaning that
        // the popup extension has called tableau.extensions.ui.closeDialog.
        $('#inactive').hide();
        $('#active').show();

        // The close payload is returned from the popup extension via the closeDialog method.
        $('#interval').text(closePayload);
        setupRefreshInterval(closePayload);
      })
      .catch((error) => {
        // One expected error condition is when the popup is closed by the user (meaning the user
        // clicks the 'X' in the top right of the dialog).  This can be checked for like so:
        switch (error.errorCode) {
          case tableau.ErrorCodes.DialogClosedByUser:
            console.log('Dialog was closed by user');
            break;
          default:
            console.error(error.message);
        }
      });
  }

  /**
   * This function sets up a JavaScript interval based on the time interval selected
   * by the user.  This interval will refresh all selected datasources.
   */
  function setupRefreshInterval (interval) {
    setInterval(function () {
      const dashboard = tableau.extensions.dashboardContent.dashboard;
      dashboard.worksheets.forEach(function (worksheet) {
        worksheet.getDataSourcesAsync().then(function (datasources) {
          datasources.forEach(function (datasource) {
            if (activeDatasourceIdList.indexOf(datasource.id) >= 0) {
              datasource.refreshAsync();
            }
          });
        });
      });
    }, interval * 60 * 1000);
  }



  /**
   * Opens the background color picker dialog.
   * Passes the current background color as the initial payload so the dialog
   * can pre-select it.  The closePayload returned is the chosen hex color,
   * which is then applied to the extension body.
   */
  function openColorDialog () {

    const popupUrl = new URL('uiNamespaceColorDialog.html', window.location.href).href;

    if (openDialogs.has(popupUrl)) {
      return; // Dialog is already open; don't open a second instance
    }

    openDialogs.add(popupUrl);

    // This checks for the selected dialog style in the radio form.
    // To send messages to the colorDialog, the colorDialog must be modeless.
    let dialogStyle;
    const dialogStyleOptions = document.getElementsByName('dialogStyleRadio');
    if (dialogStyleOptions[0].checked) {
      dialogStyle = tableau.DialogStyle.Modal;
    } else if (dialogStyleOptions[1].checked) {
      dialogStyle = tableau.DialogStyle.Modeless;
    } else {
      dialogStyle = tableau.DialogStyle.Window;
    }

    $('#sendToColorDialog').prop('disabled', false);

    tableau.extensions.ui
      .displayDialogAsync(popupUrl, currentBgColor, { height: 500, width: 360, dialogStyle: dialogStyle })
      .then((closePayload) => {
        openDialogs.delete(popupUrl);
        $('#sendToColorDialog').prop('disabled', true);
        currentBgColor = closePayload;
        $('body').css('background-color', currentBgColor);
      })
      .catch((error) => {
        openDialogs.delete(popupUrl);
        $('#sendToColorDialog').prop('disabled', true);
        if (error.errorCode === tableau.ErrorCodes.DialogClosedByUser) {
          console.log('Color dialog was closed by user');
        } else {
          console.error(error.message);
        }
      });
  }

  async function sendToColorDialog() {
    let colorDialogUrl = new URL('uiNamespaceColorDialog.html', window.location.href).href; 
    await  tableau.extensions.ui.sendDialogMessageAsync(`Hello from the extension! ${new Date().toLocaleTimeString()}`, colorDialogUrl);
  }



  /**
   * Helper that is called to set state anytime the settings are changed.
   */
  function updateExtensionBasedOnSettings (settings) {
    if (settings.selectedDatasources) {
      activeDatasourceIdList = JSON.parse(settings.selectedDatasources);
      $('#datasourceCount').text(activeDatasourceIdList.length);
    }
  }
})();


```


#### uiNamespace.trex

This example and `.trex` is for a dashboard extension, however the same principals apply to viz extensions.
If you use this code, replace the `<url>` with the location of where you are hosting your extension.
The `<configure-context-menu-item />` is required for the configuration context menu or the Format Extension button (for viz extensions). For additional popup windows or dialog boxes, no other changes are required to the `.trex` file.

```xml

<?xml version="1.0" encoding="utf-8"?>
<manifest manifest-version="0.1" xmlns="http://www.tableau.com/xml/extension_manifest">
  <dashboard-extension id="com.tableau.extensions.samples.uinamespace" extension-version="0.6.0">
    <default-locale>en_US</default-locale>
    <name resource-id="name"/>
    <description>UI Namespace Sample</description>
    <author name="tableau" email="github@tableau.com" organization="tableau" website="https://www.tableau.com"/>
    <min-api-version>1.10</min-api-version>
    <source-location>
      <url>http://localhost:8765/uinamespace.html</url>
    </source-location>
    <icon>iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAALEwAACxMBAJqcGAAAAlhJREFUOI2Nkt9vy1EYh5/3bbsvRSySCZbIxI+ZCKsN2TKtSFyIrV2WuRCJuBiJWxfuxCVXbvwFgiEtposgLFJElnbU1SxIZIIRJDKTrdu+53Uhra4mce7Oe57Pcz7JOULFisViwZ+29LAzOSjQYDgz1ZcCvWuXV11MJpN+OS/lm6179teqH0yDqxPTCyKSA8DcDsyOmOprnCaeP7459pdgy969i0LTC3IO/RQMyoHcQN+3cnljW3dNIFC47qDaK3g7BwdTkwBaBELT4ZPOUVWgKl4ZBnjxJPUlMDnTDrp0pmr6RHFeEjjcUUXPDGeSEwDN0Xg8sivxMhJNjGzbHd8PkM3eHRfkrBM5NkcQaY2vUnTlrDIA0NoaX+KLXFFlowr14tvVpqb2MICzmQcKqxvbumv+NAhZGCCIPwEw6QWXKYRL/VUXO0+rAUJiPwAk5MIlgVfwPjjHLCL1APmHN94ZdqeYN+NW/mn6I4BvwQYchcLnwFhJMDiYmlRxAzjpKWZkYkUCcZ2I61wi37tLbYyjiN0fHk5Oz3nGSLSzBbNHCF35R7f6K1/hN9PRhek11FrymfQQQKB4+Gl05P2qNRtmETlXW7e+b2z01dfycGNbfFMAbqNyKp9Jp4rzOT8RYFs0njJkc2iqsCObvTsOsDWWqA5C1uFy+Uz/oXJeKwVT4h0RmPUXhi79vuC0Ku6yOffTK3g9lfxfDQAisY516sg5kfOCiJk7HoLt2cf9b/9LANAc7dznm98PagG1fUOZ9IP5uMB8Q4CPoyNvausapkTt3rNMuvdf3C/o6+czhtdwmwAAAABJRU5ErkJggg==</icon>
    <context-menu>
      <configure-context-menu-item />
    </context-menu>
  </dashboard-extension>
  <resources>
    <resource id="name">
      <text locale="en_US">UI Namespace Sample</text>
    </resource>
  </resources>
</manifest>



```





```mdx-code-block
</TabItem>
<TabItem value="uiNcD" label="uiNamespaceColorDialog">
```

* [HTML](#uinamespacecolordialoghtml)
* [JavaScript](#uinamespacecolordialogjs)


#### uiNamespaceColorDialog.html

```html


<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Background Color</title>

    <!-- jQuery -->
    <script src="https://code.jquery.com/jquery-3.2.1.min.js"></script>

    <!-- Bootstrap -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js"></script>

    <!-- Extensions Library -->
    <script src="tableau.extensions.1.latest.js"></script>

    <!-- Our extension's code -->
    <script src="uiNamespaceColorDialog.js"></script>

    <style>
      body { padding: 20px; }
      .color-swatch {
        width: 28px;
        height: 28px;
        border-radius: 3px;
        border: 1px solid #ccc;
        margin-right: 10px;
        flex-shrink: 0;
      }
      .color-name { font-size: 14px; }
      .color-hex { font-size: 12px; color: #888; margin-left: 6px; }
      /* Dropdown: full-width trigger + menu items match former list rows */
      #colorDropdown { width: 100%; }
      #colorDropdownToggle {
        width: 100%;
        text-align: left;
        display: flex;
        align-items: center;
        justify-content: space-between;
      }
      #colorDropdownToggle .color-row-inner {
        display: flex;
        align-items: center;
        min-width: 0;
      }
      #colorDropdownToggle .caret { margin-left: 8px; flex-shrink: 0; }
      .dropdown-menu {
        width: 100%;
        max-height: 280px;
        overflow-y: auto;
        padding: 4px 0;
      }
      .dropdown-menu > li > a.color-dropdown-item {
        display: flex;
        align-items: center;
        padding: 6px 12px;
        clear: both;
        white-space: normal;
      }
      .dropdown-menu > li > a.color-dropdown-item:hover,
      .dropdown-menu > li > a.color-dropdown-item:focus {
        background-color: #f5f5f5;
      }
      .dropdown-menu > li.active > a.color-dropdown-item {
        background-color: #e8f0fe;
        outline: 2px solid #4E79A7;
        outline-offset: -2px;
      }
      #colorPreview {
        width: 100%;
        height: 60px;
        border: 1px solid #ccc;
        border-radius: 4px;
        margin-bottom: 16px;
        transition: background-color 0.2s;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h4>Background Color</h4>
      <p>Select a background color for the extension window.</p>
      <hr />

      <h5>Preview:</h5>
      <div id="colorPreview"></div>

      <h5>Tableau 10 Colors + Neutrals:</h5>
      <div class="dropdown" id="colorDropdown">
        <button type="button" class="btn btn-default dropdown-toggle" id="colorDropdownToggle" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false" aria-label="Select background color">
          <span class="color-row-inner" id="colorDropdownLabel"></span>
          <span class="caret"></span>
        </button>
        <ul class="dropdown-menu" id="colorDropdownMenu" role="menu" aria-labelledby="colorDropdownToggle"></ul>
      </div>

      <div class="container" style="border:1px solid #cecece; margin-top: 10px">
        <h6>Messages from the extension</h6>
        <div id="messageLog" style="max-height: 120px; overflow-y: auto; font-size: 0.9em;"></div>
      </div>

      <hr />
      <button id="saveButton" class="btn btn-primary">Save</button>
    </div>
  </body>
</html>

```

#### uiNamespaceColorDialog.js


```javascript

'use strict';

/**
 * Background Color Picker Dialog
 *
 * This dialog lets the user pick a background color for the parent extension
 * window from the Tableau 10 color palette plus two neutrals (12 total).
 *
 * Flow:
 *  - initializeDialogAsync receives the current background color as openPayload.
 *  - The matching color is pre-selected in the dropdown.
 *  - Clicking Save calls closeDialog with the chosen hex value as the closePayload.
 *  - The parent extension applies the color to its body background.
 */

(function () {
  // The 10 canonical Tableau 10 colors plus White and Light Gray (12 total).
  const TABLEAU_COLORS = [
    { name: 'Blue',        hex: '#4E79A7' },
    { name: 'Orange',      hex: '#F28E2B' },
    { name: 'Red',         hex: '#E15759' },
    { name: 'Teal',        hex: '#76B7B2' },
    { name: 'Green',       hex: '#59A14F' },
    { name: 'Yellow',      hex: '#EDC948' },
    { name: 'Purple',      hex: '#B07AA1' },
    { name: 'Pink',        hex: '#FF9DA7' },
    { name: 'Brown',       hex: '#9C755F' },
    { name: 'Gray',        hex: '#BAB0AC' },
    { name: 'White',       hex: '#FFFFFF' },
    { name: 'Light Gray',  hex: '#F0F0F0' },
  ];

  let selectedColor = TABLEAU_COLORS[10].hex; // default: White

  $(document).ready(function () {



    tableau.extensions.initializeDialogAsync().then(function (openPayload) {
      // openPayload is the current background color passed from the parent.
      if (openPayload && openPayload.startsWith('#')) {
        selectedColor = openPayload;
      }

      buildColorDropdown();
      updatePreview(selectedColor);

      tableau.extensions.ui.addEventListener(tableau.TableauEventType.DialogMessageReceived, (event) =>{
        console.log(`Message received: ${event.message}`);
        const p = $('<p />').text(event.message);
        $('#messageLog').append(p);
      });

      $('#saveButton').click(function () {
        sendToParent(`Message From the Color Dialog at ${new Date().toLocaleTimeString()}`);
        tableau.extensions.ui.closeDialog(selectedColor);
      });
    });
  });

  /**
   * Builds swatch + name + hex for one color (shared by trigger label and menu items).
   */
  function buildColorRowContent ($container, color) {
    $('<div />', {
      class: 'color-swatch',
      css: { backgroundColor: color.hex }
    }).appendTo($container);
    $('<span />', { class: 'color-name', text: color.name }).appendTo($container);
    $('<span />', { class: 'color-hex', text: color.hex }).appendTo($container);
  }

  /**
   * Updates the dropdown button to show the current selection.
   */
  function updateDropdownLabel (color) {
    const $label = $('#colorDropdownLabel');
    $label.empty();
    buildColorRowContent($label, color);
  }

  /**
   * Builds the Bootstrap dropdown menu and attaches selection handlers.
   */
  function buildColorDropdown () {
    const $menu = $('#colorDropdownMenu');
    $menu.empty();

    let selectedMeta = TABLEAU_COLORS.find(function (c) {
      return c.hex.toUpperCase() === selectedColor.toUpperCase();
    });
    if (!selectedMeta) {
      selectedMeta = { name: 'Custom', hex: selectedColor };
    }
    updateDropdownLabel(selectedMeta);

    TABLEAU_COLORS.forEach(function (color) {
      const isSelected = color.hex.toUpperCase() === selectedColor.toUpperCase();

      const $li = $('<li />', { role: 'presentation', class: isSelected ? 'active' : '' });
      const $a = $('<a />', {
        href: '#',
        class: 'color-dropdown-item',
        role: 'menuitem',
        tabindex: '-1'
      });

      buildColorRowContent($a, color);

      $a.on('click', function (e) {
        e.preventDefault();
        selectedColor = color.hex;
        $menu.find('li').removeClass('active');
        $li.addClass('active');
        updateDropdownLabel(color);
        updatePreview(selectedColor);
        $('#colorDropdownToggle').dropdown('toggle');
      });

      $li.append($a);
      $menu.append($li);
    });
  }

  /**
   * Updates the preview box to reflect the currently selected color.
   */
  function updatePreview (hex) {
    $('#colorPreview').css('background-color', hex);
  }

  async function sendToParent(message) {
    await tableau.extensions.ui.sendDialogMessageToParentAsync(message);
  }


})();


```

```mdx-code-block
</TabItem>
<TabItem value="uiNd" label="uiNamespaceDialog">
```


* [HTML](#uinamespacedialoghtml)
* [JavaScript](#uinamespacedialogjs)



#### uiNamespaceDialog.html



```html

<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Settings Sample</title>

    <!-- jQuery -->
    <script src="https://code.jquery.com/jquery-3.2.1.min.js"></script>

    <!-- Bootstrap -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css" >
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.3.1/js/bootstrap.min.js" ></script>

    <!-- Extensions Library (this will be hosted on a CDN eventually) -->
    <script src="tableau.extensions.1.latest.js"></script>

    <!-- Our extension's code -->
    <script src="uiNamespaceDialog.js"></script>
  </head>
  <body>
    <div class="container">
      <h4>Auto Data Source Refresh Extension</h4>
      <p>
        This Extension refreshes the selected datasources at the selected interval.
      </p>
      <hr />

      <h5>Apply to these datasources: </h5>
      <div style="margin-left: 14px" id="datasources">
      </div>
      <h5>Refresh interval in minutes:</h5>
      <input style="margin-left: 14px" type="number" id="interval" name="quantity" min="1" max="60">

      <hr />
      <button id="closeButton">Start Auto Refresh</button>
    </div>
  </body>
</html>




```

#### uiNamespaceDialog.js


```javascript

'use strict';

/**
 * UINamespace Sample Extension
 *
 * This is the popup extension portion of the UINamespace sample, please see
 * uiNamespace.js in addition to this for context.  This extension is
 * responsible for collecting configuration settings from the user and communicating
 * that info back to the parent extension.
 *
 * This sample demonstrates two ways to do that:
 *   1) The suggested and most common method is to store the information
 *      via the settings namespace.  The parent can subscribe to notifications when
 *      the settings are updated, and collect the new info accordingly.
 *   2) The popup extension can receive and send a string payload via the open
 *      and close payloads of initializeDialogAsync and closeDialog methods.  This is useful
 *      for information that does not need to be persisted into settings.
 */

// Wrap everything in an anonymous function to avoid polluting the global namespace
(function () {
  /**
   * This extension collects the IDs of each datasource the user is interested in
   * and stores this information in settings when the popup is closed.
   */
  const datasourcesSettingsKey = 'selectedDatasources';
  let selectedDatasources = [];

  $(document).ready(function () {
    // The only difference between an extension in a dashboard and an extension
    // running in a popup is that the popup extension must use the method
    // initializeDialogAsync instead of initializeAsync for initialization.
    // This has no affect on the development of the extension but is used internally.
    tableau.extensions.initializeDialogAsync().then(function (openPayload) {
      // The openPayload sent from the parent extension in this sample is the
      // default time interval for the refreshes.  This could alternatively be stored
      // in settings, but is used in this sample to demonstrate open and close payloads.
      $('#interval').val(openPayload);
      $('#closeButton').click(closeDialog);

      const dashboard = tableau.extensions.dashboardContent.dashboard;
      const visibleDatasources = [];
      selectedDatasources = parseSettingsForActiveDataSources();

      // Loop through datasources in this sheet and create a checkbox UI
      // element for each one.  The existing settings are used to
      // determine whether a datasource is checked by default or not.
      dashboard.worksheets.forEach(function (worksheet) {
        worksheet.getDataSourcesAsync().then(function (datasources) {
          datasources.forEach(function (datasource) {
            const isActive = selectedDatasources.indexOf(datasource.id) >= 0;

            if (visibleDatasources.indexOf(datasource.id) < 0) {
              addDataSourceItemToUI(datasource, isActive);
              visibleDatasources.push(datasource.id);
            }
          });
        });
      });
    });
  });

  /**
   * Helper that parses the settings from the settings namespace and
   * returns a list of IDs of the datasources that were previously
   * selected by the user.
   */
  function parseSettingsForActiveDataSources () {
    let activeDatasourceIdList = [];
    const settings = tableau.extensions.settings.getAll();
    if (settings.selectedDatasources) {
      activeDatasourceIdList = JSON.parse(settings.selectedDatasources);
    }

    return activeDatasourceIdList;
  }

  /**
   * Helper that updates the internal storage of datasource IDs
   * any time a datasource checkbox item is toggled.
   */
  function updateDatasourceList (id) {
    const idIndex = selectedDatasources.indexOf(id);
    if (idIndex < 0) {
      selectedDatasources.push(id);
    } else {
      selectedDatasources.splice(idIndex, 1);
    }
  }

  /**
   * UI helper that adds a checkbox item to the UI for a datasource.
   */
  function addDataSourceItemToUI (datasource, isActive) {
    const containerDiv = $('<div />');

    $('<input />', {
      type: 'checkbox',
      id: datasource.id,
      value: datasource.name,
      checked: isActive,
      click: function () {
        updateDatasourceList(datasource.id);
      }
    }).appendTo(containerDiv);

    $('<label />', {
      for: datasource.id,
      text: datasource.name
    }).appendTo(containerDiv);

    $('#datasources').append(containerDiv);
  }

  /**
   * Stores the selected datasource IDs in the extension settings,
   * closes the dialog, and sends a payload back to the parent.
   */
  function closeDialog () {
    tableau.extensions.settings.set(datasourcesSettingsKey, JSON.stringify(selectedDatasources));

    tableau.extensions.settings.saveAsync().then((newSavedSettings) => {
      tableau.extensions.ui.closeDialog($('#interval').val());
    });
  }
})();



```

```mdx-code-block
</TabItem>
</Tabs>
```