# CapacitorJs-in-deez-nuts
Understand how CapacitorJS work in nut shell.

## Plugin in Java

```java
@CapacitorPlugin(name = "HolyPipe")
public class HolyPipePlugin extends Plugin {
    @PluginMethod(returnType = PluginMethod.RETURN_NONE)
    public void on(PluginCall feOn) {}

    @PluginMethod(returnType = PluginMethod.RETURN_CALLBACK)
    public void emit(PluginCall call) {}
}
```

## Auto-generated script will be injected to WebBrowser by JsInjector (every requests).

```javascript
(function(w) {
var a = (w.Capacitor = w.Capacitor || {});
var p = (a.Plugins = a.Plugins || {});
var t = (p['HolyPipe'] = {});
// predefined 
t.addListener = function(eventName, callback) {
  return w.Capacitor.addListener('HolyPipe', eventName, callback);
}
t['removeAllListeners'] = function(_options) {
    return w.Capacitor.nativeCallback('HolyPipe', 'removeAllListeners', _options)
}
t['checkPermissions'] = function(_options) {
    return w.Capacitor.nativePromise('HolyPipe', 'checkPermissions', _options)
}
t['requestPermissions'] = function(_options) {
    return w.Capacitor.nativePromise('HolyPipe', 'requestPermissions', _options)
}

// 2 more method which we declare by @PluginMethod
// emit returnType === PluginMethod.RETURN_CALLBACK
t['emit'] = function(_options, _callback) {
    return w.Capacitor.nativeCallback('HolyPipe', 'emit', _options, _callback)
}
// on returnType === PluginMethod.RETURN_NONE
t['on'] = function(_options) {
    return w.Capacitor.nativeCallback('HolyPipe', 'on', _options)
}
})(window);
```

### parts of window.Capacitor object which will be injected to WebBrowser in every request
```
window.Capacitor.addListener = (pluginName, eventName, callback) => {
  const callbackId = cap.nativeCallback(pluginName, 'addListener', { eventName: eventName }, callback);
  return {
    remove: async () => {
      var _a;
      (_a = win === null || win === void 0 ? void 0 : win.console) === null || _a === void 0 ? void 0 : _a.debug('Removing listener', pluginName, eventName);
      cap.removeListener(pluginName, callbackId, eventName, callback);
    },
  };
};

window.Capacitor.removeListener = (pluginName, callbackId, eventName, callback) => {
  cap.nativeCallback(pluginName, 'removeListener', { callbackId: callbackId, eventName: eventName }, callback);
};

window.Capacitor.nativeCallback = (pluginName, methodName, options, callback) => {
  if (typeof options === 'function') {
    console.warn(`Using a callback as the 'options' parameter of 'nativeCallback()' is deprecated.`);
    callback = options;
    options = null;
  }
  return cap.toNative(pluginName, methodName, options, { callback });
};

window.Capacitor.toNative = (pluginName, methodName, options, storedCallback) => {
  var _a, _b;
  try {
    if (typeof postToNative === 'function') {
      let callbackId = '-1';
      if (storedCallback && (typeof storedCallback.callback === 'function' || typeof storedCallback.resolve === 'function')) {
        // store the call for later lookup
        callbackId = String(++callbackIdCount);
        callbacks.set(callbackId, storedCallback);
      }
      const callData = {
        callbackId: callbackId,
        pluginId: pluginName,
        methodName: methodName,
        options: options || {},
      };
      if (cap.isLoggingEnabled && pluginName !== 'Console') {
        cap.logToNative(callData);
      }
      // post the call data to native
      postToNative(callData);
      return callbackId;
    }
    else {
      (_a = win === null || win === void 0 ? void 0 : win.console) === null || _a === void 0 ? void 0 : _a.warn(`implementation unavailable for: ${pluginName}`);
    }
  }
  catch (e) {
   (_b = win === null || win === void 0 ? void 0 : win.console) === null || _b === void 0 ? void 0 : _b.error(e);
  }
  return null;
};

// entry point to receive json data from Native.
window.Capacitor.fromNative = result => {
  var _a, _b;
  if (cap.isLoggingEnabled && result.pluginId !== 'Console') {
    cap.logFromNative(result);
  }
  
  // get the stored call, if it exists
  try {
    const storedCall = callbacks.get(result.callbackId);
    if (storedCall) {
      // looks like we've got a stored call
      if (result.error) {
        // ensure stacktraces by copying error properties to an Error
        result.error = Object.keys(result.error).reduce((err, key) => {
          // use any type to avoid importing util and compiling most of .ts files
          err[key] = result.error[key];
          return err;
        }, new cap.Exception(''));
      }
     
      if (typeof storedCall.callback === 'function') {
        // callback
        if (result.success) {
          storedCall.callback(result.data);
        }
        else {
          storedCall.callback(null, result.error);
        }
      }
      else if (typeof storedCall.resolve === 'function') {
        // promise
        if (result.success) {
          storedCall.resolve(result.data);
        }
        else {
          storedCall.reject(result.error);
        }
        // no need to keep this stored callback
        // around for a one time resolve promise
        callbacks.delete(result.callbackId);
      }
    }
    else if (!result.success && result.error) {
      // no stored callback, but if there was an error let's log it
      (_a = win === null || win === void 0 ? void 0 : win.console) === null || _a === void 0 ? void 0 : _a.warn(result.error);
    }
    if (result.save === false) {
      callbacks.delete(result.callbackId);
    }
  }
  catch (e) {
    (_b = win === null || win === void 0 ? void 0 : win.console) === null || _b === void 0 ? void 0 : _b.error(e);
  }
  // always delete to prevent memory leaks
  // overkill but we're not sure what apps will do with this data
  delete result.data;
  delete result.error;
};

// entry point to send data to native. available by MessageHandler.postMessage method below.
postToNative = data => {
  var _a;
  try {
    win.androidBridge.postMessage(JSON.stringify(data));
  }
  catch (e) {
    (_a = win === null || win === void 0 ? void 0 : win.console) === null || _a === void 0 ? void 0 : _a.error(e);
  }
};
```

### Message transfer
```java
public class MessageHandler {
    private Bridge bridge;
    private WebView webView;
    private PluginManager cordovaPluginManager;

    public MessageHandler(Bridge bridge, WebView webView, PluginManager cordovaPluginManager) {
        this.bridge = bridge;
        this.webView = webView;
        this.cordovaPluginManager = cordovaPluginManager;
        webView.addJavascriptInterface(this, "androidBridge");
    }

    /**
     * The main message handler that will be called from JavaScript
     * to send a message to the native bridge.
     * @param jsonStr
     */
    @JavascriptInterface
    @SuppressWarnings("unused")
    public void postMessage(String jsonStr) {
        try {
            JSObject postData = new JSObject(jsonStr);

            String type = postData.getString("type");

            boolean typeIsNotNull = type != null;
            boolean isCordovaPlugin = typeIsNotNull && type.equals("cordova");
            boolean isJavaScriptError = typeIsNotNull && type.equals("js.error");

            String callbackId = postData.getString("callbackId");

            if (isCordovaPlugin) {
                String service = postData.getString("service");
                String action = postData.getString("action");
                String actionArgs = postData.getString("actionArgs");

                Logger.verbose(
                    Logger.tags("Plugin"),
                    "To native (Cordova plugin): callbackId: " +
                    callbackId +
                    ", service: " +
                    service +
                    ", action: " +
                    action +
                    ", actionArgs: " +
                    actionArgs
                );

                this.callCordovaPluginMethod(callbackId, service, action, actionArgs);
            } else if (isJavaScriptError) {
                Logger.error("JavaScript Error: " + jsonStr);
            } else {
                String pluginId = postData.getString("pluginId");
                String methodName = postData.getString("methodName");
                JSObject methodData = postData.getJSObject("options", new JSObject());

                Logger.verbose(
                    Logger.tags("Plugin"),
                    "To native (Capacitor plugin): callbackId: " + callbackId + ", pluginId: " + pluginId + ", methodName: " + methodName
                );

                this.callPluginMethod(callbackId, pluginId, methodName, methodData);
            }
        } catch (Exception ex) {
            Logger.error("Post message error:", ex);
        }
    }
    
    private void callPluginMethod(String callbackId, String pluginId, String methodName, JSObject methodData) {
        PluginCall call = new PluginCall(this, pluginId, callbackId, methodName, methodData);
        bridge.callPluginMethod(pluginId, methodName, call);
    }

    private void callCordovaPluginMethod(String callbackId, String service, String action, String actionArgs) {
        cordovaPluginManager.exec(service, action, callbackId, actionArgs);
    }

    // entry point to send from native to web browser
    public void sendResponseMessage(PluginCall call, PluginResult successResult, PluginResult errorResult) {
        try {
            PluginResult data = new PluginResult();
            data.put("save", call.isKeptAlive());
            data.put("callbackId", call.getCallbackId());
            data.put("pluginId", call.getPluginId());
            data.put("methodName", call.getMethodName());

            boolean pluginResultInError = errorResult != null;
            if (pluginResultInError) {
                data.put("success", false);
                data.put("error", errorResult);
                Logger.debug("Sending plugin error: " + data.toString());
            } else {
                data.put("success", true);
                if (successResult != null) {
                    data.put("data", successResult);
                }
            }

            boolean isValidCallbackId = !call.getCallbackId().equals(PluginCall.CALLBACK_ID_DANGLING);
            if (isValidCallbackId) {
                final String runScript = "window.Capacitor.fromNative(" + data.toString() + ")";
                final WebView webView = this.webView;
                webView.post(() -> webView.evaluateJavascript(runScript, null));
            } else {
                bridge.getApp().fireRestoredResult(data);
            }
        } catch (Exception ex) {
            Logger.error("sendResponseMessage: error: " + ex);
        }
        if (!call.isKeptAlive()) {
            call.release(bridge);
        }
    }
}

public class PluginCall {
  ...
  public void resolve(JSObject data) {
    PluginResult result = new PluginResult(data);
    this.msgHandler.sendResponseMessage(this, result, null);
  }
  ...
}
```

### Usage in WebBrowser

```javascript
// usage from web
const {remove} = HolyPipe.addListener('error', cb);
cap.nativeCallback(pluginName, 'addListener', { eventName: 'error' }, cb) 

HolyPipe.on(_options, cb)
cap.nativeCallback('HolyPipe', 'on', _options, _callback) 
```
