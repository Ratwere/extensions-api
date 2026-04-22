---
title: Add a Configuration Popup Dialog Box
description: How to add a configuration dialog box to the extension
---

If you want users to be able to configure settings for your extension, you can use an optional callback function when you initialize your dashboard or viz extension. The callback function creates a configuration option that can be used to open a popup window (dialog box) for your extension. The Extensions API supports different dialog box styles: modal, modeless, or window. You can use the dialog box to allow users to set and save settings for the extension. Starting with Tableau 2026.1 (and the v1.16 library), you can create multiple dialog boxes and send messages between them. 

:::info

By design, there can be only one context configuration popup window or dialog box. For information about creating and using multiple pop dialog boxes or window, see [Add a Multiple Popup Dialog Boxes](trex_multiple_dialogs.md).

:::

## Add the context menu to the `.trex` file

The first step is to add the `<context-menu>` option to the extension's manifest file (`.trex`). The `<context-menu>` element only contains one item: `<configure-context-menu-item />`.

* The context menu option must follow the `<icon>` and `<permissions>` elements in the manifest file in the `<dashboard-extension>` or `<worksheet-extension>` section:

```xml
<!-- add to <dashboard-extension> or <worksheet-extension> section
  after <icon></icon> 
  and <permissions></permissions> -->

<context-menu>
    <configure-context-menu-item />
</context-menu>

```

## Create a configuration function

When you initialize an extension, you can pass an optional `contextMenuCallbacks` object to the `initializeAsync()` function.
This object maps a special ID or key (which must be `'configure'`) to a function you create.  

* For dashboard extensions, the function you create, in conjunction 
with adding a `<context-menu>` item to the manifest, adds a new **Configure...** context menu item to the zone of the extension inside a dashboard.

* For viz extensions, the function you create, in conjunction 
with adding a `<context-menu>` item to the manifest, adds a new **Format Extension** button to the Marks card.  

When the user selects the context menu item, or selects **Format Extensions** button, the configuration function you specified is executed.

**Dashboard extensions configuration menu**

![](../assets/extension_configure_menu.png)

----

**Viz extensions format button**

![](../assets/viz_format_btn_75.png)
  
For example, you could use the UI namespace and have the configuration function call the `displayDialogAsync()` function. The function then creates a dialog box that can be used to change settings for the extension. The parent (or initial window) for your extension might have the following JavaScript code. This example uses an initial payload string value, *defaultIntervalInMin*, to pass to the configuration dialog. The payload value is modified in the configuration dialog and is returned in the `closeDialog()` method. Alternatively, you could use a `Settings` object to store the key/value pairs that configure your extension.

The `popupURL` is the URL to your configuration window. You will need to create the HTML web page and JavaScript code for the configuration window.

```javascript

// Wrap everything in an anonymous function to avoid polluting the global namespace
(function () {
const defaultIntervalInMin = '5';

$(document).ready(function () {
    // ...
    // pass the object to initializeAsync() to map 'configure' key to a function called configure()
    // ...
    tableau.extensions.initializeAsync({'configure': configure}).then(function() {     
      // ...
     // ... code to set up event handlers for changes to configuration 
      });
    });
  


   function configure() { 
    // ... code to configure the extension
    // for example, set up and call displayDialogAsync() to create the configuration window 
    // and set initial settings (defaultIntervalInMin)
    // and handle the return payload 

    // Define the URL of the configuration popup window
    // This uses the window.location.origin property to retrieve the scheme, hostname, and
    // port where the parent extension is currently running, so this string doesn't have
    // to be updated if the extension is deployed to a new location.
    const popupUrl = `${window.location.origin}/Samples/Dashboard/UINamespace/uiNamespaceDialog.html`;
    // 
    // Specify the style of dialog box: modal, modeless, or window
    let dialogStyle = tableau.DialogStyle.Modeless;
    // ...
    // initial payload string value, `defaultIntervalInMin` set 
    tableau.extensions.ui.displayDialogAsync(popupUrl, defaultIntervalInMin, { height: 500, width: 500, dialogStyle }).then((closePayload) => {
      // The promise is resolved when the dialog has been expectedly closed, meaning that
      // the popup extension has called tableau.extensions.ui.closeDialog.
      // ...

      // The close payload is returned from the popup extension via the closeDialog() method.
     // ....

    }).catch((error) => {
      //  ... 
      // ... code for error handling
    });
  }

})();  

```

## Create the HTML and JavaScript code for your configuration window

In the JavaScript code for the popup dialog window, add your code for initializing the dialog (`initializeDialogAsync`) and code for setting and saving the configuration settings. Include this JavaScript file in your HTML code for the configuration window.

```javascript

  $(document).ready(function () {
    // The only difference between an extension in a dashboard and an extension
    // running in a popup is that the popup extension must use the method
    // initializeDialogAsync instead of initializeAsync for initialization.
    // This has no affect on the development of the extension but is used internally.
    tableau.extensions.initializeDialogAsync().then(function (openPayload) {
    // The openPayload sent from the parent extension, in this example, is the
    // default time interval for the refreshes.  This could alternatively be stored
    // in settings, but is used in this sample to demonstrate open and close payloads.
    // code goes here
    });
  }); 

```

In your code to close the popup window, you must pass a *non-empty* string value as the return payload, even if you're using the `Settings` object to pass your configuration parameters. For example, you could specify a string with a single blank space (`" "`) as the return payload.

```javascript
  function closeDialog() {
   // Save the settings with tableau.extensions.settings.saveAsync()
   // Or pass the new configuration setting in the close payload
     tableau.extensions.ui.closeDialog('NewInterval');
     console.log("Settings saved");
    });

```

To better understand how to use the context menu or configuration window, and to see it in action, check out the [UINamespace](https://github.com/tableau/extensions-api/tree/master/Samples/Dashboard/UINamespace?=target="_blank") sample.

## Create multiple popup dialog boxes or windows 

Starting with Tableau 2026.1, you can create more than one popup window or dialog box for your extension. In addition to the optional configuration popup that you can create when you initialize the extension, you can have your extension open other dialog boxes, by adding buttons and controls to your extension. If you are creating multiple dialog boxes, you need to consider how you want them to interact with each other and the user. You can control whether the popup windows are modal or modeless, and whether they need to share information and payloads. If you have multiple popup windows, you'll need to track which popup window is active, so you can control their interactions.

### The default configuration window

By design, there can be only one context configuration item. For dashboard extensions, this is the popup dialog box that appears when users select the **Configure..** option from the context menu. For viz extensions, this is the dialog box that appears when users select the **Format Extension** button on the Marks card. 

The configuration item is specified in the `.trex` file for the extension. 

```xml
<!-- add to <dashboard-extension> or <worksheet-extension> section
  after <icon></icon> 
  and <permissions></permissions> -->

<context-menu>
    <configure-context-menu-item />
</context-menu>

```

And by specifying a callback function (`configure()`) that gets called when the `'configure'` option is specified in the call to initialize the extension (`initializeAsync()`).

```javascript
  $(document).ready(function () {
    tableau.extensions.initializeAsync({'configure': configure}).then(function() {
    // When the user clicks the Configure... context menu item,
    // or Format Extension, the configure() function specified 
    // as the argument here is executed.
    //
   });

```

If you are creating additional popup windows, you'll need to call those popup windows from other methods, and you'll need to add controls to call those methods. For example, you could add a button or toolbar item in the main extension window, and use that button to open another popup window. The content and logic of the dialog boxes or windows are typically in HTML and JavaScript files that are separate from the main extension files.

### Create additional popup windows

Starting in Tableau 2026.1, you can create multiple popup windows. While you can't display these additional dialog boxes or windows when the user selects the configure context item, you can display them when users click buttons and controls in your extension. 

* Determine how you want the pop windows to interact (if at all). Should they be modal or modeless? Do they need to share information or content? 

* To create additional dialog boxes or popup windows, add controls (buttons, toolbars) to your extension. 

* Create the HTML and JavaScript code that define the content and features of your additional popup windows. 

* In your extension JavaScript or TypeScript code, define your popup windows: the `popupURL`, the dialog style, and dimensions, and any initial payload you want to send to the popup window. Pass in these values when you call the `displayDialogAsync()` method.

Like the configuration popup, the URL of the additional popup windows must belong to the same domain as the parent extension. The relative paths must resolve to the directory, or a child directory, of the extension. Root-relative paths are not allowed. For example, `./config.html` or `config.html` are allowed, but not the root-relative path `/config.html`.

For example, you could add another dialog box to the uiNamespace sample that lets users set the background color of the extension. In the `uiNamespace.html` file, you could add a button to display the popup window. When the user clicks the button, a color formatting dialog opens. The user can select the background color. When the user clicks a Save button, it calls the `closeDialog()` method and closes the popup window, which returns the new background color as the payload and it is applied. 

```html
<button id="openColorDialog" class="btn btn-secondary" style="margin-top: 10px; margin-left: 6px">Set Background Color</button>
```

In the `uiNamespace.js` file, you would add code to set the default background color and then call the `displayDialogAsync()`. You would need to create the HTML and Javascript code for the popup window (for example, uiNamespaceColorDialog.html and uiNamespaceColorDialog.js). 

```javascript

/**
* Sets the default background color
* Sets the current color to match as the initial payload
*/

  const defaultBgColor = '#FFFFFF';
  let currentBgColor = defaultBgColor;

/**
* Opens the background color picker dialog (always modal).
* Passes the current background color as the initial payload so the dialog
* can pre-select it.  The closePayload returned is the chosen hex color,
* which is then applied to the extension body.
*/

  function openColorDialog () {
    const popupUrl = new URL('uiNamespaceColorDialog.html', window.location.href).href;

    tableau.extensions.ui
      .displayDialogAsync(popupUrl, currentBgColor, { height: 500, width: 360, dialogStyle: tableau.DialogStyle.Modal })
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







## Send messages between configuration windows and the extension