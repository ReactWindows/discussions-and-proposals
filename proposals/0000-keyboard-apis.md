---
title: Keyboard events
author: Harini Kannan, Stephen Peters
date: 05/21/2019
---

# RFC0000: Keyboard APIs

## Summary

Components need the ability to handle key strokes and implement custom logic when interacting with a React Native app using Keyboard. This proposal outlines the basic props and events needed to enable this scenario.

## Motivation

There are several usecases where end users may interact with `react-native` applications using keyboard. These include:
- iPads and Android tablet devices with attached keyboards
- `react-native-windows` apps that will be deployed on PCs and laptop computers
- `react-native-web` apps that could be used in any end point including Chromebooks, PCs and tablet devices 

In the realm of enabling keyboard interactions, there are 2 sets of scenarios that motivate the need for Keyboarding APIs:
1. The `react-native` JS components including lean-core components like Button, TouchableOpacity, TouchableHighlight, FlatList etc., and other community modules that are authored in JS (not using any direct underlying native control) will need to be enhanced to support default keyboard interactions. For example: firing Button.onPress() when the end user hits *Enter* key.
2. Developers may want to handle key strokes themselves to provide any custom interactions before the underlying platform consumes the keystroke for any default activity. 

Both the above scenarios are motivations to surface a basic set of Keyboarding APIs as outlined in the sections below.

### Scope

This proposal deals with the minimal set of APIs needed to achieve some of the simplest keyboarding support. The following are not in scope for this proposal and can be added on later if there are scenarios demanding them:
- `onKeyPress` events for IME/auto-complete scenarios where each character that is received needs to be processed independently. 

## Basic example

### Example 1 : Simple key stroke handling

In the following example, the lastKeyDown prop will contain the key stroke from the end user when keyboard focus is on View. 
```
  <View onKeyDown={this._onKeyDown} />
      
  private _onKeyDown = (event: IKeyboardEvent) => {
    this.setState({ lastKeyDown: event.nativeEvent.key });
  };
  
```

### Example 2 : Custom key stroke handling in native components

In the following example, the app's logic takes precedence when keystrokes are encountered in the TextInput before the native platform can handle them. 
```
  <TextInput onKeyUp={this._onKeyUp} />
  
  private _onKeyUp = (event: IKeyboardEvent) => {
    if(event.nativeEvent.key == 'Enter'){
            //do something custom when Enter key is pressed
         }
  };
```

## Detailed design

The APIs being introduced here will follow the models created by :
- The corresponding React APIs for [Keyboarding events](https://reactjs.org/docs/events.html#keyboard-events)
- The corresponding Windows APIs for [Keyboarding events](https://docs.microsoft.com/en-us/windows/uwp/design/input/keyboard-events)

### Keyboard events
The following events will be introduced on `View` and `TextInput` to cover the most common use cases where key stroke handling is likely to occur. Other individual components where they may be neeeded can wrap a `View` around to capture the Key events.

| API | Args | Returns | Description |
|:---:|:----:|:-------:|:----:|
| onKeyDown | IKeyboardEvent | void | Occurs when a keyboard key is pressed when a component has focus. On Windows, this corresponds to [KeyDown](https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.uielement.keydown)|
| onKeyDownCapture | IKeyboardEvent | void | Occurs when the `onKeyDown` event is being routed. `onKeyDown` is the corresponding bubbling event. On Windows, this corresponds to [PreviewKeyDown](https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.uielement.previewkeydown)|
| onKeyUp | IKeyboardEvent | void | Occurs when a keyboard key is released when a component has focus. On Windows, this corresponds to [KeyUp](https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.uielement.keyup)|
| onKeyUpCapture | IKeyboardEvent | void | Occurs when the `onKeyUp` event is being routed. `onKeyUp` is the corresponding bubbling event. On Windows, this corresponds to [PreviewKeyUp](https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.uielement.previewkeyup)|

Where `IKeyboardEvent` will be a new event type added to `ReactNative.NativeSyntheticEvents` of type `INativeKeyboardEvent`. `INativeKeyboardEvent` is a new interface and will expose the following properties:

| Property | Type | Description |
|:---:|:----:|:----:|
| key | string | The character typed by the user. |
| altKey | boolean | The `Alt` (Alternative) key. Also maps to Apple `Option` key. |
| ctrlKey | boolean | The `Ctrl` (Control) key. |
| shiftKey | boolean | The `Shift` key. |
| metaKey | boolean | Maps to Windows `Logo` key and the Apple `Command` key. |
| eventPhase | KeyEventPhase | Current phase of routing for the key event. |

Where `KeyEventPhase` is a new enum to detect whether the keystroke is being tunneled/bubbled to the target component that has focus. It has the following fields:

- None : none
- Capturing : when the keydown/keyup event is being captured while tunneling its way from the root to the target component
- AtTarget : when the keydown/keyup event has reached the target component that is handling the corresponding event
- Bubbling : when the keydown/keyup event is being captured while bubbling its way to the parent(s) of the target component

### Declarative properties

To co-ordinate the handoffs of these Keyboard events between the native layer and the JS layer, we are also introducing 2 corresponding properties on `View` and `TextInput` components. These are:

| Property | Type | Description |
|:---:|:----:|:----:|
| keyDownEvents | IHandledKeyboardEvents[] | Specifies the key(s) that are handled in the JS layer by the onKeyDown/onKeyDownCapture events |
| keyUpEvents | IHandledKeyboardEvents[] | Specifies the key(s) that are handled in the JS layer by the onKeyUp/onKeyUpCapture events |

Where `IHandledKeyboardEvents` is a new type which takes a string parameter named `key` to declare the key strokes that are of interest to the JS layer.

When the `onKeyXX` events are handled by the app code, the corresponding native component will have KeyXX events marked as *handled* for the declared key strokes.

### TBD

- Threading model details and other more specific implementation details

## Drawbacks

The primary use-case for `react-native` is mobile apps on iOS and Android where keyboard may not be super important as an input device. Hence this may be seen as additional complexity in API surface. However, this will not be impacting any existing apps/code. End users using keyboard (even in iOS/Android) will automatically see keyboard interactions in lean-core components light up. Developers who choose to use the new APIs will be able to take advantage of the additional keyboarding capabilities. Those who choose not to will see no change in the behavior of their existing code. This should be overall goodness as the scope for `react-native` usage grows beyond just mobile applications.

When implementing for each platform, different keys may map to different keyCodes which is why the `INativeKeyboardEvent` interface has been kept as a simple one with only one string property in lieu of separate key parameters for Ctrl, Alt etc., 

## Alternatives

- Introduce a separate `KeyEventHandlerComponent` that can be wrapped around any React Native component to handle keystrokes. This was rejected as it feels too clunky and separate where basic keyboarding needs to be a part of `View` so all apps can get these behaviors and APIs by default.
- Introduce the APIs and implement default keystroke handling in *windows* forked custom components like `ViewWindows` or `ButtonWindows`. This was rejected because the default behavior of react-native components need to have basic keyboard behaviors in Windows and developers bringing crossplatform React Native code should have their Views *just work* on Windows and Web. Also, there may be usecases even in iOS/Andoid with attached keyboards where these APIs and default behaviors could be handy.

## Adoption strategy

This will be a new API and not a breaking change. This is being implemented in the `react-native-windows` out-of-tree platform first to validate the APIs and implementations. Once vetted, we propose to add this to `react-native` and add documentation in the official API documentation.

## How we teach this

These APIs should be presented as a continuation of React patterns. As such, it should be very familiar to existing web/React developers who can relate these APIs to concepts they already know.

Once implemented, these APIs should be documented as part of official `react-native` API documentation. 

## Unresolved questions

- Focus APIs are coming up next for review and will be needed for the full end to end story to be complete for keyboarding.
