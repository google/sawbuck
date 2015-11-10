Sawbuck uses the same set of [tools as does Chrome/Chromium](http://www.chromium.org/developers/how-tos/build-instructions-windows). Assuming you're set up with Visual Studio 2005 or 2008 and the depot tools, to build Sawbuck you do as follows:

Create a new directory, and create this .gclient file in there:
```
solutions = [
  { "name"        : "src",
    "url"         : "https://sawbuck.googlecode.com/svn/trunk",
    "custom_deps" : {
    },
    "safesync_url": ""
  },
]
```

You can also do this by something like:
```
gclient config https://sawbuck.googlecode.com/svn/trunk --name src
```

In a command shell with the above directory as the current working directory, run "gclient sync". This will fetch all the sources and create your project files. Now open the solution file "src\sawbuck\sawbuck.sln" in Visual Studio and build the project.

This should build the sawbuck binary, the test logger and the sawbuck MSI file.