Description of **Sawdust**, a background ETW log collector for Windows, intended to aid in diagnosing bugs in the wild. Logs collected by _Sawdust_ can be viewed and analyzed using _Sawbuck_.

# Introduction #

_Sawdust_ is a companion to _Sawbuck_. It is a Windows systray application, unobtrusive and lightweight. Its sole function is to collect logs from configured providers and then upload then to a crash server. Sawdust is not included with Sawbuck installer, you have to install it separately.


# How to use _Sawdust_ #

1. Install using the setup program shared on this site.
2. Start _Sawdust_. An icon will appear on the system tray.
3. Hover over the icon, make sure logs are being collected.
4. Use your software as usual; attempt to reproduce problem you are after.
5. Once you managed that, upload log data to the crash server by selecting the appropriate command from the program's context menu (through the SysTray icon).

# Configuring _Sawdust_ #

Sawdust comes with a default configuration file in JSON format. You can edit this file to define:
1. Which programs (providers) are traced (by default it is Chrome and related things).
2. How logs are stored (how much space is allotted and so on).
3. What should happen when 'Upload' command is selected. By default, logs are compressed and uploaded to Google's crash server. However, you can instead have these logs uploaded to another server (following the same protocol) or to a LAN location.