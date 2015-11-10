**DEPRECATED: Please use the [new version](https://code.google.com/p/syzygy/wiki/PythonBuild) of the page.**


#summary Python build and develoment process.


At build time, we effectively create a new python deployment in the Debug/Release directory (through virtualenv magic) and stock it with the external modules we rely on.

Our modules and scripts are then built to eggs which are placed in the Debug/Release directory.
To use the scripts or modules, after building them, you can do something like this:

```
> Debug\py\Scripts\easy_install benchmark-chrome```

This will install the benchmark module and its dependencies to the python installation.
To subsequently use the script, you can do:

```
> Debug\py\Scripts\benchmark <parameters>```

During development, it may be easier to install the scripts and modules in the python environment using development mode. Development mode leaves the scripts in-place, but uses setuptools magic to munge the python installation's path to include them.

There's a little helper script in the syzygy directory called "develop\_py.bat", which will install all our modules and scripts in develop mode, or you can do this per-module by hand with incantations like this:
```

> Debug\py\Scripts\activate
(py) > python scripts\benchmark\setup.py develop
(py) > python scripts\benchmark\benchmark.py <parameters>
```

The "activate" invocation will change your path, so that the correct python interpreter is the default, and the "develop" command to setup.py will install the benchmark module in development mode - e.g. in-place to the interpreter.

For any future scripts we create, please make sure you write a setup.py. Use setuptools, and add build configuration to make sure they're built to egg on the continuous builder, as otherwise they tend to bitrot.

## Using Eclipse & PyDev ##

It's easy to use Eclipse's PyDev with this setup.
Build the "virtualenv" target in the Syzygy solution file, then invoke the "develop\_py.bat" script.
Next, setup a new python interpreter in the PyDev preferences by pointing it at the python instance "Debug\py\scripts\python.exe". You probably want to select all the paths suggested, save perhaps for the module/script you're going to work on.
After this you can create a PyDev project that includes the files you're going to work on, and PyDev should be able to resolve all the system and Syzygy modules you use.