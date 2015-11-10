**Sawbuck** is a log viewer and controller for Windows Chrome logging, and for other applications or plugins that use the [logging facility in Chrome base](http://src.chromium.org/viewvc/chrome/trunk/src/base/logging.h?view=markup).

Logging in Chrome is integrated with Event Tracing for Windows (ETW), which allows ETW controllers like Sawbuck to control log verbosity at runtime.
The Chrome logging integration also captures the call stack at the logging site, which can then be resolved and displayed by log viewers such as Sawbuck. This is especially handy for plugin code, such as e.g. Chrome Frame.

## Screenshot ##
![http://sawbuck.googlecode.com/files/Sawbuck_screenshot.png](http://sawbuck.googlecode.com/files/Sawbuck_screenshot.png)

## How to get this for my own code? ##

If you're using the Chrome base library, then integrating your logging with ETW is as simple as allocating a new GUID by running [guidgen.exe](http://msdn.microsoft.com/en-us/library/ms241442(VS.80).aspx), then initializing the ETW log provider as so.

```
#include "base/logging.h"
#include "base/logging_win.h"
#include <initguid.h>

// {????????-????-????-????-????????}
DEFINE_GUID(kMyLogProvider,
    0x????????,
    0x????,
    0x????,
    0x??, 0x??, 0x??, 0x??, 0x??, 0x??, 0x??, 0x??);

void InitLogging() {
  logging::LogEventProvider::Initialize(kMyLogProvider);
}

```

You will then need to write your provider GUID and name to the registry for Sawbuck to know about it. Create a new key at
```
HKEY_LOCAL_MACHINE\Software\Google\Sawbuck\Providers
```
with your provider GUID as the name of the key, with your provider textual name as the default string value of the key.
Feel free to send a patch to the Sawbuck installer with your changes to the [installer's registration](http://code.google.com/p/sawbuck/source/browse/trunk/sawbuck/installer/sawbuck.wxs#83) to get your provider added to Sawbuck.

If you're not using Chrome base, you'll need to duplicate what [chrome/base/logging\_win.cc](http://src.chromium.org/viewvc/chrome/trunk/src/base/logging_win.cc?revision=37164&view=markup) does.

## How to Contribute ##

Start by [downloading an installer](http://code.google.com/p/sawbuck/downloads/list), installing it and playing with Sawbuck.
Browse and perhaps sign up to the [sawbuck-dev discussion group](http://groups.google.com/group/sawbuck-dev).
Get the [sources and build them](http://code.google.com/p/sawbuck/wiki/HowToBuild).
Check out the [issues list](http://code.google.com/p/sawbuck/issues/list).
[Contribute a patch](http://code.google.com/p/sawbuck/wiki/HowToContribute).