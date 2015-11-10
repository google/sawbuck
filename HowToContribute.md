Sawbuck uses the same tools and the same [codereview process as Chrome](http://dev.chromium.org/developers/contributing-code).
You might start by taking on one of the filed issues, post on the [sawbuck-dev](http://groups.google.com/group/sawbuck-dev) mailing list to find a reviewer, then upload your change for review.

To build Sawbuck, you first need to install the [Chromium depot tools](https://sites.google.com/a/chromium.org/dev/developers/how-tos/depottools).
You then need to create a file named ".gclient" in a new directory, and populate it with the following:
```
solutions = [
  { "name"        : "src",
    "url"         : "http://sawbuck.googlecode.com/svn/trunk",
    "custom_deps" : {
    },
    "safesync_url": "",
  },
]
```

After this is done, you run `gclient sync` in the directory, which should fetch the Sawbuck sources and all dependencies.
The gclient tool also creates the project solution files, so after syncing you should be able to open the solution files at
  1. "src\sawbuck\sawbuck.sln" or
  1. "src\syzygy\sysygy.sln".

If you have integrated ETW logging in your own project, and would like to register a new provider, note that the provider registration is done by the installer. See
http://code.google.com/p/sawbuck/source/browse/trunk/sawbuck/installer/sawbuck.wxs#140.