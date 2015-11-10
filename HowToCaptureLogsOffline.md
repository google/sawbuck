# Introduction #

Event Tracing for Windows has command line tools that allow you to set up event tracing sessions to log to files which can be later imported to Sawbuck for viewing and analysis.

# Details #

The [logman.exe](http://technet.microsoft.com/en-us/library/cc753820(WS.10).aspx) utility ships with all recent Windows systems, and it can be used to capture trace logs for offline viewing with Sawbuck.

To start capturing an offline log, you need to do something like the following:

**startlog.bat**:
```
:: The -ct argument is not supported on XP, so avoid using it there.
setlocal
set clock_type=-ct cycle
ver | findstr "XP" > nul
if not %ERRORLEVEL% == 0 goto main
set clock_type=

:main
:: Create a kernel logger session and make it log image events to the file kernel.etl.
:: You only need this session if you want to lookup symbols for the traces later.
logman create trace -ets "NT Kernel Logger" %clock_type% -mode globalsequence -o kernel.etl -p "Windows Kernel Trace" (img,process)
:: Create the logger session.
logman create trace -ets "Log Session" %clock_type% -mode globalsequence -o log.etl
:: Turn on the Chrome, ChromeFrame and Sawbuck providers.
logman update trace -ets "Log Session" -p {0562BFC3-2550-45B4-BD8E-A310583D3A6F} 0xffffffff 4
logman update trace -ets "Log Session" -p {C43B1318-C63D-465B-BCF4-7A89A369F8ED} 0xffffffff 4
logman update trace -ets "Log Session" -p {7FE69228-633E-4F06-80C1-527FEA23E3A7} 0xffffffff 4
:: And the Chrome Perf provider
logman update trace -ets "Log Session" -p {3DADA31D-19EF-4dc1-B345-037927193422} 0xffffffff 4
```

To stop the log, you can do something like the following:

**stoplog.bat**:
```
:: Stop the kernel logger.
logman stop -ets "NT Kernel Logger"
:: Stop the log session.
logman stop -ets "Log Session"
```

After this you need to import the files **kernel.etl** and **log.etl** into Sawbuck.