# NodeRT: Use WinRT in Node, Electron, and NW.js

:computer: Example
```
npm install --save @nodert-win10-rs3/windows.ui.notifications
```

| SDK | Known As | Windows Version | npm Scope |
| --- | --- | --- | --- |
| Windows 10, Build 17134 | April 2018 Update (Redstone 4) | 1803 | [npmjs.com/org/nodert-win10-rs4](https://www.npmjs.com/org/nodert-win10-rs4) |
| Windows 10, Build 16299 | Fall Creators Update (Redstone 3) | 1709 |  [npmjs.com/org/nodert-win10-rs3](https://www.npmjs.com/org/nodert-win10-rs3) |
| Windows 10, Build 15063 | Creators Update (Redstone 2) | 1703 | [npmjs.com/org/nodert-win10-cu](https://www.npmjs.com/org/nodert-win10-cu) |
| Windows 10, Build 14393 | Anniversary Update (Redstone 1) | 1607 | [npmjs.com/org/nodert-win10-au](https://www.npmjs.com/org/nodert-win10-au) |
| Windows 10, Build 10586 | Threshold 2 | 1511 | [npmjs.com/~nodert-win10](https://www.npmjs.com/~nodert-win10) |

For an in-depth overview, [check out our `//build` talk][talk].

In general, any WinRT/UWP API that can be called by a desktop app can by called by
Node.js or Electron using NodeRT. There are notable exceptions, but UWP APIs
are [generally callable from desktop applications][desktop-callable].

For more examples of what NodeRT can do, check out the [samples](./samples).

## :books: Documentation

We've split the documentation up into two parts. If you want to use NodeRT modules,
read on. If you want to use NodeRT to compile your own WinRT modules, [see
the module creation guide](./MODULE_CREATION.md).

 * [Using NodeRT modules](#using-nodert-modules)
 * [Naming and Properties](#nodert-modules-naming-and-properties)
 * [Namespaces](#namespaces)
 * [Class inheritance and object casting](#class-inheritance-and-object-casting)
 * [Using WinRT streams in Node.js](#using-winrt-streams-in-node.js)
 * [Building and Consuming in Electron](#nodert-and-electron)
 * [License](#license)
 * [Attributions](#attributions)
 * [Contribute](#contribute)


### Using NodeRT Modules

NodeRT automatically exposes Microsoft’s UWP/WinRT APIs to the Node.js environment
by generating Node modules for all Windows namespaces. This enables Node.js developers
to write code that consumes native Windows capabilities. The generated modules' APIs
are (almost) the same as the [WinRT APIs listed on MSDN][uwp-namespaces].

As an example, let's check out the `Windows.Devices.Geolocation` namespace to
locate the user from Node.js.

:one: First, ensure that you have the Windows 10 SDK installed. In this example,
we're using the [Fall Creators Update SDK][sdk-archive].
:two: Then, install `@nodert-win10-rs3/windows.devices.geolocation`. The npm
organization denotes the used SDK version - in this case, it's the Fall Creators
Update, which has the codename "Redstone 3".

```sh
npm i --save @nodert-win10-rs3/windows.devices.geolocation
```

Once you've installed the module, you're ready to use your computer's geolocation
features. In this example, we're creating a new `Geolocator` and are calling
its instance method [`getGeopositionAsync()`][getgeopositionasync].

```js
const { Geolocator } = require('windows.devices.geolocation')
const locator = new Geolocator()

locator.getGeopositionAsync((error, result) => {
  if (error) {
    console.error(error)
    return
  }

  const { coordinate } = result
  const { longitude, latitude } = coordinate

  console.info(longitude, latitude)
})
```

Let's take a closer look at the whole module:

![Windows.Devices.Geolocation NodeRT module contents](/doc/images/object_contents.png)

As you can see, you can create new WinRT objects using the `new` operator. In
order to inspect the method and properties of the object, you can inspect
its prototype. For example, a new `Geolocator` object looks like this:

```javascript
console.log(new geolocation.Geolocator().__proto__)
```
And the output will be:

![Geolocator prototype contents](/doc/images/golocation__proto.png)

> :memo: Note that property values are fetched on the fly, and hence have `undefined`
> values when printing the prototype.

### Naming and Properties

NodeRT uses the same JavaScript conventions as Microsoft does for
JavaScript-based UWP applications.

 * Class/Enum names have the first letter in upper-case
 * Class/Enum fields (properties, methods, and events) always start with a
   lower-case letter. The remainder of the name is identical to what you'd
   find on MSDN.
 * Enums are JavaScript objects with keys corresponding to the enum
   fields and values to the enum field's numeric values.

#### Properties

Properties on WinRT objects behave like JavaScript properties.

```js
locator.reportInterval = 2000
console.info(locator.reportInterval)
```

#### Synchronous methods

Straight-forward JavaScript: Make the call with the appropriate arguments
and don't even worry about the fact that you're calling native code. If
there are several WinRT overloads for the method, make the call with the
right set of arguments and the correct overload of the method will be
called:

```js
const { XmlDocument }  = require('windows.data.xml.dom')

const xmlDoc = new xml.XmlDocument();
xmlDoc.loadXml('<node>some text here</node>')
```

#### Asynchronous methods

Asynchronous method accepts the same variables as listed on MSDN - with the
addition of a completion callback as the last argument.

This callback will be called when the function has finished, and will receive
an error as the first argument, and the result as the second argument:

```js
locator.getGeopositionAsync((err, result) => {
  if (error) {
    console.error(error)
    return
  }

  // Result is of type "Geoposition"
  const { coordinate } = result
  const { longitude, latitude } = coordinate

  console.info(longitude, latitude)
})
```

#### Events

Registering to events is done using the class' `on` method (which is equivalent
to `addListener`), which receives the event name (case insensitive) and the
event handler function.

For example:

```js
const handler = (sender, eventArgs) => {
  console.info('status is:', eventArgs.status)
}

locator.on('statusChanged', handler)
```

Unregistering from an event is done the same way, using the class's `off` or
`removeListener` methods.

```js
// Using same event handler as in the example above
locator.off('statusChanged', handler);
```

### Namespaces

Each NodeRT module represents a single namespace. For instance, `windows.storage`
is its own NodeRT module - and `windows.storage.streams` is another NodeRT module.

The reason for this separation is strictly due to performance considerations.
Separating the code allows you to load only the code you actually intend to
use, meaning that Node.js won't fill the machine's memory.

This architecture means that in case you are using a NodeRT module which
contains a function, property, or event which returns an object from another
namespace, then you will need to require that namespace **before** calling
that function/property/event.

For example:

```js
const capture = require('windows.media.capture')

// We also require this module in order to be able to access
// device controller properties
const devices = require('windows.media.devices')

const capture = new capture.MediaCapture()

capture.initializeAsync((error, result) => {
  if (error) {
    return console.error(error);
  }

  // Get the device controller, its type (VideoDeviceController) is defined in
  // the windows.media.devices namespace. That's why we had to load
  // windows.media.devices before calling this method.
  const deviceController = capture.videoDeviceController

  // We can now use the VideoDeviceController regularly
  deviceController.brightness.trySetValue(-1)
})
```

### Class inheritance and object casting

Since some WinRT classes inherit from other classes. You might also need to cast an
object of a certain type to another type.

In order to do so, each NodeRT object has a static method named <b>castFrom</b>
which accepts another object and tries to cast it to the class' type. Magical, right?

The following example casts an [IXmlNode][IXmlNode] object to an
[XmlElement][XmlElement]:

```js
const xml = require('windows.data.xml.dom')

...

// Obtain a list of nodes:
const nodesList = ....
const xmlNode = nodesList.getAt(0)

// Cast xmlNode to XmlElement
const xmlElement = xml.XmlElement.castFrom(xmlNode)

// We can now use XmlElement functions
xmlElement.setAttribute('attr', 'value')
```

### Using WinRT streams in Node.js

In order to support the use of WinRT streams in Node.js, we have created the
[`nodert-streams`][streams] module, which bridges between WinRT streams and
Node.js streams.

This bridge enable the conversion of WinRT streams to Node.js streams, such
that WinRT streams could be used just as regular Node.js streams.

### NodeRT and Electron

NodeRT modules are native Node addons. As such, you'll need to compile
them for usage with Electron. If you're using an Electron boilerplate
or CLI (like `electron-forge` or `electron-builder`), the embedded tools
will automatically compile the native code correctly.

If you are not using a boilerplate or are having trouble correctly compiling
NodeRT modules, see Electron's documentation for [how to use native Node addons
in Electron](https://electronjs.org/docs/tutorial/using-native-node-modules).

From Electron v14 on, you'll have to require NodeRT modules in the `main` process.
See https://github.com/NodeRT/NodeRT/issues/158 for details.

## License

NodeRT is released under the Apache 2.0 license.
For more information, please take a look at the <a href="./LICENSE">license</a> file.

## Attributions

In order to build NodeRT we used these 2 great libraries:
* RazorTemplates - https://github.com/volkovku/RazorTemplates
* RX.NET - https://github.com/Reactive-Extensions/Rx.NET/

## Contribute
You are welcome to send us any bugs you may find, suggestions, or any other
comments. Before sending anything, please go over the repository issues list,
just to make sure that it isn't already there.

You are more than welcome to fork this repository and send us a pull request
if you feel that what you've done should be included.

[talk]: https://channel9.msdn.com/Events/Build/2017/T6976
[desktop-callable]: https://docs.microsoft.com/en-us/windows/desktop/apiindex/uwp-apis-callable-from-a-classic-desktop-app
[uwp-namespaces]: http://msdn.microsoft.com/en-us/library/windows/apps/br211377.aspx
[geolocation]: http://msdn.microsoft.com/library/windows/apps/br225603
[sdk-archive]: https://developer.microsoft.com/en-us/windows/downloads/sdk-archive
[getgeopositionasync]: https://docs.microsoft.com/en-us/uwp/api/windows.devices.geolocation.geolocator.getgeopositionasync
[IXmlNode]: https://docs.microsoft.com/en-us/uwp/api/Windows.Data.Xml.Dom.IXmlNode
[XmlElement]: https://docs.microsoft.com/en-us/uwp/api/Windows.Data.Xml.Dom.XmlElement
[streams]: https://github.com/NodeRT/nodert-streams
