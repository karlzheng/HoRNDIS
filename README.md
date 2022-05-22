# HoRNDIS(the USB tethering driver for Mac OS X)

**HoRNDIS** (pronounce: *"horrendous"*) is a driver for Mac OS X that allows you to use your Android phone's native [USB tethering](http://en.wikipedia.org/wiki/Tethering) mode to get Internet access.

For more information, [visit the home page for HoRNDIS on my site](http://www.joshuawise.com/horndis).

## Installation

### From Source/Binary

* Get the installation package ([Download](http://www.joshuawise.com/horndis) or [Build](#building-the-source) the installation package from source yourself)
* Run the installation package

### From Homebrew

```sh
brew cask install horndis
sudo kextload /Library/Extensions/HoRNDIS.kext
```

## Configuration

* Assuming that the installation proceeds without errors, after it completes, connect your phone to your Mac by USB.
* Enter the settings menu on your phone.
* In the connections section, below Wi-Fi and Bluetooth:
  * Select "More..."
  * Select "Tethering & portable hotspot"
* Check the "USB tethering" box. It should flash once, and then become solidly checked.

## Uninstallation

* Delete the `HoRNDIS.kext` under `/System/Library/Extensions` and `/Library/Extensions` folder
* Restart your computer

## Building the source

* `git clone` the repository
* Simply running xcodebuild in the checkout directory should be sufficient to build the kext.
* If you wish to package it up, you can run `make` to assemble the package in the build/ directory

## Debugging and Development Notes

This sections contains tips and tricks for developing and debugging the driver.

### USB Device Information

*Mac OS System Menu* -> *About This Mac* -> *System Report* --> *Hardware*/*USB* <br>
Lists all USB devices that OS recognizes. Unfortunately, it does not give USB descriptors.

`lsusb -v`<br>
It prints USB configuration, such as interface and endpoint descriptors. You can print it for all devices or limit the output to specific ones. In order to run this command, you need to install *usbutils*.
* Homebrew users: `brew install mikhailai/misc/usbutils`<br>
  Please *do not* install *lsusb* package from Homebrew Core, it's a different utility with the same name.
* Macports users: `sudo port install usbutils`

### IO Registry

`ioreg -l -r -c IOUSBHostDevice`<br>
This command lists all the Mac OS IO Registry information under all USB devices. Unlike *lsusb*, ioreg tells how Mac OS recognized USB devices and interfaces, and how it matched drivers to these interfaces. The `-r -c IOUSBHostDevice` limits the output to USB devices; to get complete OS registry, please run `ioreg -l`.

### OS Logging

The `LOG(....)` statements, sprinkled throughout the HoRNDIS code, call the `IOLog` functions. On Mac OS *El Capitan* (10.11) and earlier, the log messages go into `/var/log/system.log`. Starting from *Sierra* (10.12), these messages are no longer written to `system.log` and instead can be viewed via:
* *GUI*, using *Console* application, located in *Utilities* folder. You need to enter `process:kernel` in the search box in order to filter the relevant messages.
* *Command Line*, using the `log` command. For example:<br>
  `log show --predicate process==\"kernel\" --start "2018-12-11 22:54:00"`<br>
  The start value needs to be several minutes in the past, so it would not flood the console. Here is a convenient command that prints the messages from the past 3 mintes:<br>
  `log show --predicate process==\"kernel\" --start "$(date -v-3M +'%F %T')"`

I've observed that Mac OS logging is unreliable (especially in *Sierra*). In some cases, the messages may come out garbled (looking like bad multi-threaded code). In other cases, either GUI or Command Line may be missing messages that were emitted. Sometimes, reloading the driver may fix the problem.





## Compiling the driver for run on Apple Silicon (M1) macOS 12 Monterey

Thanks [akemin-dayo](https://github.com/akemin-dayo) at  [https://github.com/jwise/HoRNDIS/issues/146](https://github.com/jwise/HoRNDIS/issues/146) said,

I'm glad to report that the latest version of HoRNDIS (9.2) works perfectly on Apple Silicon machines with no code changes required!

Basically, just compiling an additional arm64e (not arm64) binary slice to the HoRNDIS kext works! [@jwise](https://github.com/jwise)

Tested on macOS 12.0.1 21A559, "Monterey".

(I do realise that this isn't really an *issue* per se, but I just felt really bad for all the users complaining about broken functionality here…)

(Plus, speaking from my own personal experience as a fellow developer, I *do* find it quite helpful when someone else already tested compatibility for me ;P)

------

##### If you're a general user coming who just wants to use HoRNDIS on Apple Silicon right this instant…

Ever since macOS / OS X 10.10 Yosemite, Apple has required kernel extensions (kexts) to be signed by developers who are subscribed to the Apple Developer Program, who *also* have to apply for a special kext signing privilege.

You're… probably not one of those people. ;P

Luckily, there *is* a way for you to sign your own kexts using an Xcode feature called ad-hoc signing! But it does require changing some settings first.

##### Switching to "Reduced Security" mode

If you've *already* placed your Mac in "Reduced Security" mode before, simply skip this section.

1. Shut down your Apple Silicon Mac.
2. Press and hold down the power button *until* the text under the Apple logo says "Loading startup options…", then let go.
3. Select "Options".
4. You are now in recoveryOS — enter your password if it asks.
5. Go to Utilities → Startup Security Utility.
6. Select "Reduced Security" and enable Allow user management of kernel extensions from identified developers".
7. Shut down your Apple Silicon Mac.

##### Disabling SIP (System Integrity Protection)

**IMPORTANT:** Disabling SIP in *any* capacity, even partially, will also disable Apple Pay, as well as any iOS-on-macOS apps you may have downloaded from the App Store. This is a strange (and annoying) decision that Apple has decided to make specifically on Apple Silicon, as Apple Pay actually works fine even when SIP is disabled on x86_64 (Intel) Macs.

1. Follow steps 2〜4 from above.
2. Go to Utilities → Terminal.
3. Type in the following to *fully* disable SIP: `csrutil disable`
   **Note:** It is possible to only *partially* disable the part of SIP that enforces kext signature verification (`csrutil enable --without kext`), but according to Apple, this is apparently an "unsupported configuration". Use it if you wish (as many do already), but please make sure to read and fully understand the warning that `csrutil` gives if you try.
4. Reboot your Apple Silicon Mac.

##### Compiling HoRNDIS for Apple Silicon (arm64e)

1. Download and install [Xcode](https://apps.apple.com/app/xcode/id497799835).
2. Run the following in a Terminal session. When it asks for your password, it is normal for no characters to show when when you type!

```
git clone --recursive https://github.com/jwise/HoRNDIS.git
cd Development/HoRNDIS/
xcodebuild -sdk macosx -configuration Release
sudo cp -rv build/Release/HoRNDIS.kext /Library/Extensions/
```

1. Go to System Preferences → Security & Privacy and approve the HoRNDIS kernel extension.
2. Reboot, connect an Android device in USB tethering mode, and enjoy using HoRNDIS again!

