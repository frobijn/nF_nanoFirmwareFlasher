# Changes to support the controlled update versioning strategy

## Motivation

The current version of the .NET **nanoFramework** is based on "auto-update everything" versioning: make sure that you always use the latest version of NuGet packages, firmware and tools, and you can be sure that the packages and tools are consistent with each other.

This may be fine if there *never* are breaking changes in the .NET **nanoFramework**, and if it is *always* possible to update the firmware on any device. But that is definitely not the case. A recent move to littlefs was a breaking change: it would have broken some low-level libraries of this contributor (File.GetLastWriteTime disappeared) and made deployed firmware unusable. As OTA-deployment of firmware is not yet available, this contributor would have a hard time updating all hard-to-reach devices. Maybe this contributor just wanted to update the application to fix a small bug - forget it. First do a major rework of the libraries, than retrieve and update all devices, and only after all that work it is possible to fix the bug. Put on the hat of a project manager and it is obvious that this "auto-update everything" strategy is ill suited for a lot of more challenging projects.

What is needed is a versioning strategy where the user of the framework (and not the framework's core team) is in charge of updating the framework components used by the project. In this **controlled update** strategy the user works with a frozen version of the framework and has a guarantee very early in the development process that the final deployment to a device will succeed. The user decides if and when to update to a next version of the framework.

## Documentation

The auto-update and the controlled update strategies are described in a [getting started guide](https://github.com/frobijn/nF_nanoframework.github.io/blob/Versioning/content/getting-started-guides/getting-started-versioning.md).

The use of a firmware archive is added to [Packaging, versioning and deployment](https://github.com/frobijn/nF_nanoframework.github.io/blob/Versioning/content/architecture/deployment.md) in the architecture section.

## New framework features

- It should be possible to use a frozen version of the framework. This can already been done for most of the packages and tools:

	- *nanoff* and *nanoclr* can be installed as local tools and will not auto-update by themselves.
	- NuGet packages: the assumption is that old versions will be available for a long time. If a user is concerned that packages disappear, NuGet has sufficient features to create a local cache of all relevant packages and use that instead of nuget.org. No need to have additional features in nanoFramework.
	- The test platform v3 can use a local *nanoclr* without auto-update.

	Still missing and to be implemented as new features are:

	- *nanoff* always uses the online cloudsmith repository. It should be possible (just like nuget) to create and use a local cache instead. This is done by adding extra options to *nanoff*.
	- The Device Explorer in Visual Studio always uses the global nanoclr and has no option to use a local version. This is done by adding an extra option to the Virtual Device tab of the Device Explorer. The data for this extra option should come from a configuration file (`nano.devices.json`, see next item).

- The framework should offer a tool to check the consistency of the NuGet packages and firmware packages that in the end will be deployed to a device. The check is now done when deploying an application from Visual Studio or even after a deployment to a device with *nanoff*, but that is far too late. Ideally the tool should perform the check immediately after a NuGet package is added to a project. This is implemented via:

	- A MSBuild task that runs after each build and verifies whether the versions in the AssemblyNativeVersion-attribute of class libraries (added via a NuGet package) matches the native assembly versions as used to create the firmware of the devices the code will ultimately be deployed to.
	
	- A set of configuration files (`nano.devices.json`) that inform the MSBuild task which firmware will be used. This includes the firmware of the devices that are used in testing (e.g., the Virtual Device).

- The MSBuild task should take up as little execution time as possible. That implies that the information on the native assemblies embedded in a runtime should be readily available. At the moment it is only possible to get that information by connecting a nanoFramework debug engine to a device that is connected to a serial port. That takes up too much time, and requires real hardware that may not yet exist. Instead it should be possible to get the list in other ways:

	- The list of native assemblies and their versions should be present in the firmware package that is downloaded from cloudsmith.
	- For the runtime of the Virtual Device, the list should be obtainable by running nanoclr.exe with a (new) command line option.

- Bonus feature! If we introduce a `nano.devices.json` that informs the nanoFramework components what device types and serial ports are important, then this should be the only place a user of the framework should provide that information:

    - The test platform also needs to know about local versions of *nanoclr* or of its runtime. The test platform v3 should use `nano.devices.json` instead of having the configuration in .runsetting files. The .runsetting files should be reserved for test-specific configurations. The test platform v2 should stay as it is for backward compatibility.

    - The test platform v3 also has an option to specify serial ports that should not be touched. There is overlap with the `nano.devices.json` where ports are excluded because they are running (a virtual device with) the wrong version of the firmware. It would be better to make the exclusion list a part of the `nano.devices.json` configuration so that other nanoFramework tools (like the Device Explorer) use the same information.

- Exclusive access to a device. 

    - The test platform v3 needs a protection that the same device (connected to a serial port) cannot be accessed by several test hosts at the same time. It is better to move that protection to the nf-debugger library, so that the protection is used by all nanoFramework tools. The required changes are part of a separate [PR](https://github.com/nanoframework/nf-debugger/pull/376). The implementation of the *controlled update versioning* features should be based on the results of that PR.

    - The test platform v3 also needs protection that there are no concurrent updates to the same *nanoclr.exe* tool, so that for updates there should be an exclusive access to the *nanoclr.exe* tool. This feature should be part of the *controlled update versioning* implementation and not exclusively of the test platform. The Device Explorer should also use that protection.

# Impacted repositories

The implementation of the features is distributed over multiple repositories:

- nanoframework.github.io for the update of the documentation.
- nanoFirmwareFlasher for features concerning the the firmware archive.
- nf-interpreter for features concerning the list of native assemblies.
- nf-Visual-Studio-extension for features concerning the Device Explorer.

The MSBuild task and the code to read its configuration is added to the nf-Visual-Studio-extension repository, as that repository already has some build tasks, and because the code to read the configuration has to be shared by the MSBuild task and the Visual Studio extension. There are no other code dependencies for the new features between the repositories.

# Implementation notes

## nanoframework.github.io

- The auto-update and the controlled update strategies are described in a [getting started guide](https://github.com/frobijn/nF_nanoframework.github.io/blob/Versioning/content/getting-started-guides/getting-started-versioning.md).

- The use of a firmware archive is added to [Packaging, versioning and deployment](https://github.com/frobijn/nF_nanoframework.github.io/blob/Versioning/content/architecture/deployment.md) in the architecture section.

## nanoFirmwareFlasher

The changes in this repository implement the firmware archive feature:

- Extra command line options added to *nanoFirmwareFlasher.Tool* and the implementation to *nanoFirmwareFlasher.Library* to download cloudsmith packages to a firmware archive, list the content of the archive, and deploy firmware from the archive to a device.

- Apart from firmware .zip packages, the .dll for the *WIN_DLL_nanoCLR* target can be downloaded and is placed in the archive in such a way if can be used as runtime for *nanoclr.exe*.

- Added a description of the new command line options to the README.md file

- Made some changes to *nanoFirmwareFlasher.Tool* and *nanoFirmwareFlasher.Library* so that the code can be tested:
    - *nanoFirmwareFlasher.Tests* is a friend of *nanoFirmwareFlasher.Tool* and *nanoFirmwareFlasher.Library*.
    - Use of new OutputWriter instead of Console. The OutputWriter's output can be collected per unit test even if they are executed in parallel
    - The location of the firmware archive can be set in unit tests

- Added unit tests to *nanoFirmwareFlasher.Tests* to test the affected code.

- Added .editorconfig. This may have resulted in some changes to *nanoFirmwareFlasher.Tool* and *nanoFirmwareFlasher.Library*: update of the file header, use of explicit type instead of `var`.

- **Still TODO**: use nf-debugger's serial-port protection.

## nf-interpreter

The changes in this repository have been made to get a list of names and versions of the native assemblies that are used in a runtime, without having to connect to a device via a serial port and retrieve the list via the wire protocol.

- NanoCLR.exe: an option has been added: --nativeassemblies, similar to (and can be combined with) --getversion. The result is a list of names, version and checksums sent to the standard output.

    It calls a method in the CLR host assembly to get information about the native assemblies (a .NET class). The data for that class is deserialized from data obtained via two additional methods in the native NanoCLR runtime that serializes the static constants in the runtime with native assembly information. If a tool requires the list of native assemblies, it can run the nanoclr.exe and parse its output.

- NanoCLR.exe: added --clrinstancepath to use with --nativeassemblies, so that the above keeps working if nanoclr.exe is updated but an older runtime is used.

- CMake/Modules/FindNF_NativeAssemblies.cmake: create a `native_assemblies.csv` file in the build output directory of a firmware package that contains the names, version and checksums of the native assemblies selected to be part of the firmware.

    - The CMake code to extract version and checksum works well for the native assemblies that use the skeleton generated by the metadata processor. There are two native assemblies that define the checksum in a different way:
        - src\nanoFramework.Runtime.Events\nf_rt_events_native_nanoFramework_Runtime_Events_EventSink.cpp has a g_CLR_AssemblyNative_nanoFramework_Runtime_Events_EventSink_DriverProcs with a #define as checksum (DRIVER_INTERRUPT_METHODS_CHECKSUM). However, this is not the checksum of the native assembly corresponding to the nanoFramework.Runtime.Events .NET class library - that is located in src\nanoFramework.Runtime.Events\nf_rt_events_native.cpp. This exception can be ignored for our purpose.
        - src\CLR\CorLib\corlib_native.cpp: The checksum depends on NANOCLR_REFLECTION that corresponds to TARGET_NANOCLR_REFLECTION in CMake.As the corlib is already treated as a special (as the corlib is always included and treated differently in CMake), the parsing is also done differently.

    - File creation has been tested for a few firmware packages. As there does not seem to be a target with `NF_FEATURE_SUPPORT_REFLECTION = OFF`, that part of the CMake script could not be tested. The CMake regex has been tested by temporarily using it for the `NF_FEATURE_SUPPORT_REFLECTION = ON` case.
    - As .csv files are copied to the firmware .zip package, it is assumed the `native_assemblies.csv` will become part of the firmware package. This is not tested.
    - **Still TODO**: what is needed to create a new version of all firmware packages so that `native_assemblies.csv` will become part of the reference/community packages?

- Added .editorconfig.

## nf-Visual-Studio-extension

A new library / NuGet package is introduced: `nanoFramework.Targeting.Tooling`, with unit tests in `nanoFramework.Targeting.Tooling.Tests`. This is designed to contain all logic for the various nanoFramework tools. It targets .NET 6 and .NET Framework 4.7.2 (same as nf_debugger library). It contains an implementation of the features:

- Read hierarchy of `nano.devices.json` configuration.
- Device discovery based on the `nano.devices.json` configuration.
- Install / update / exclusive access to *nanoclr.exe*.
- Obtain list of native assemblies and versions from a local firmware package, nanoclr/nanoclr runtime, connected device, .NET nanoFramework assemblies.
- Consistency check of required and available native assemblies.

A new MSBuild task and NuGet package is introduced: `nanoFramework.Versioning`. Available in .NET 6 (dotnet build) and .NET Framework 4.7.2 (VS MSBuild) version. No unit tests as all features are implemented in `nanoFramework.Targeting.Tooling`.

Also added .editorconfig.
