Sawbuck is a log viewer and controller for Windows Chrome logging, and for other applications or plugins that use the logging facility in Chrome base.

Logging in Chrome is integrated with Event Tracing for Windows (ETW), which allows ETW controllers like Sawbuck to control log verbosity at runtime. The Chrome logging integration also captures the call stack at the logging site, which can then be resolved and displayed by log viewers such as Sawbuck. 

How to get this for my own code?
If you're using the Chrome base library, then integrating your logging with ETW is as simple as allocating a new GUID by running guidgen.exe, then initializing the ETW log provider as so.

```C++
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

HKEY_LOCAL_MACHINE\Software\Google\Sawbuck\Providers
with your provider GUID as the name of the key, with your provider textual name as the default string value of the key. Feel free to send a patch to the Sawbuck installer with your changes to the installer's registration to get your provider added to Sawbuck. If you're not using Chrome base, you'll need to duplicate what chrome/base/logging_win.cc does.

How to Contribute
Start by downloading an installer, installing it and playing with Sawbuck. Browse and perhaps sign up to the [sawbuck-dev](https://groups.google.com/forum/#!forum/sawbuck-dev) discussion group. Get the sources and build them. Check out the issues list. Contribute a patch. Have fun!
