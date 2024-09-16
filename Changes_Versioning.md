# Changes to support the controlled update versioning strategy

## Motivation

The current version of the .NET **nanoFramework** is based on "auto-update everything" versioning: make sure that you always use the latest version of NuGet packages, firmware and tools, and you can be sure that the packages and tools are consistent with each other.

This may be fine if there *never* are breaking changes in the .NET **nanoFramework**, and if it is *always* possible to update the firmware on any device. But that is definitely not the case. A recent move to littlefs was a breaking change: it would have broken some low-level libraries of this contributor (File.GetLastWriteTime disappeared) and made deployed firmware unusable. As OTA-deployment of firmware is not yet available, this contributor would have a hard time updating all hard-to-reach devices. Maybe this contributor just wanted to update the application to fix a small bug - forget it. First do a major rework of the libraries, than retrieve and update all devices, and only after all that work it is possible to fix the bug. Put on the hat of a project manager and it is obvious that this "auto-update everything" strategy is ill suited for a lot of more challenging projects.

What is needed is a versioning strategy where the user of the framework (and not the framework's core team) is in charge of updating the framework components used by the project. In this **controlled update** strategy the user works with a frozen version of the framework and has a guarantee very early in the development process that the final deployment to a device will succeed. The user decides if and when to update to a next version of the framework.

## Documentation

The auto-update and the controlled update strategies are described in a [getting started guide](https://github.com/frobijn/nF_nanoframework.github.io/blob/Versioning/content/getting-started-guides/getting-started-versioning.md).

The guide also describes how to configure the new v3 test platform for this scenario. That platform has not yet been released.

## New framework features

- It should be possible to use a frozen version of the framework. This can already been done for most of the packages and tools:

	- *nanoff* and *nanoclr* can be installed as local tools and will not auto-update by themselves.
	- NuGet packages: the assumption is that old versions will be available for a long time. If a user is concerned that packages disappear, NuGet has sufficient features to create a local cache of all relevant packages and use that instead of nuget.org. No need to have additional features in nanoFramework.
	- The test platform v3 can use a local *nanoclr* without auto-update.

	Still missing and to be implemented as new features are:

	- *nanoff* always uses the online cloudsmith repository. It should be possible (just like nuget) to create and use a local cache instead. This is done by adding extra options to *nanoff*.
	- The Device Explorer in Visual Studio always uses the global nanoclr and has no option to use a local version. This is done by adding an extra option to the Virtual Device tab of the Device Explorer. The data for this extra option should come from a configuration file (`nano.devices.json`, see next item).

- The framework should offer a tool to check the consistency of the NuGet packages and firmware packages that in the end will be deployed to a device. The check is now done when deploying an application from Visual Studio or even after a deployment to a device with *nanoff*, but that is far too late. The tool should perform the check immediately after a NuGet package is added to a project. This is implemented via:

	- A MSBuild task that runs after each build and verifies whether the versions in the AssemblyNativeVersion-attribute of class libraries (added via a NuGet package) matches the native assembly versions as used to create the firmware of the devices the code will ultimately be deployed to.
	
	- A set of configuration files (`nano.devices.json`) that informs the MSBuild task which firmware to use, and where the local copies of the tools are.

- At the moment it is only possible to get a list of native assembly versions by connecting a nanoFramework debug engine to a device that is connected to a serial port. That is not very helpful. Instead it should be possible to get the list in other ways:

	- The list of native assemblies and their versions should be present in the firmware package that is downloaded from cloudsmith.
	- For the runtime of the Virtual Device, the list should be obtainable by running nanoclr.exe with a (new) command line option.

# Impacted repositories

The implementation of the features is distributed over multiple repositories:

- nanoframework.github.io for the update of the documentation.
- nf-interpreter for features concerning the list of native assemblies.
- nanoFirmwareFlasher for features concerning the the firmware archive.
- nf-Visual-Studio-extension for features concerning the Device Explorer.

The MSBuild task and the code to read its configuration is added to the nf-Visual-Studio-extension repository, as that repository already has some build tasks, and because the code to read the configuration has to be shared by the MSBuild task and the Visual Studio extension. There are no other code dependencies for the new features between the repositories.

# Implementation notes

- NanoCLR.exe: an option has been added: --nativeassemblies, similar to (and can be combined with) --getversion. It calls a method in the CLR host assembly to get information about the native assemblies (a .NET class). The data for that class is deserialized from data obtained via two additional methods in the native NanoCLR runtime that serializes the static constants in the runtime with native assembly information. If a tool requires the list of native assemblies, it can run the nanoclr.exe and parse its output.