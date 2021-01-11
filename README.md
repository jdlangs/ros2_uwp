# ros2_uwp
Instructions, tools and examples for incorporating ROS2 in Windows UWP
Application Containers

## Motivation

The [Universal Windows Platform (UWP)](https://docs.microsoft.com/en-us/windows/uwp/get-started/universal-application-platform-guide)
is the preferred technology for creating modern applications for all versions
of Windows 10. They're also the *only* way to create and deploy applications
for certain Windows devices such as [IoT Core](https://developer.microsoft.com/en-us/windows/iot/)
or [HoloLens](https://www.microsoft.com/en-us/hololens) on which robotic
applications are particularly interesting.

[ROS2](https://index.ros.org/doc/ros2/) is the newest version of ROS developed
as a cross-platform standard desktop framework. With some work, ROS2 can be
made to work within the context of a UWP application. This repository provides
instructions, tools and examples to aid the process of building a version of
ROS2 that's compatible with the WinRT/UWP API.

## What's different about ROS2 in UWP?

ROS2 is ordinarily built as a standard Desktop application targeting the full
Win32 API. UWP applications run in a sandboxed context (for security) and are
allowed use of the [WinRT APIs](https://github.com/MicrosoftDocs/winrt-api)
as well as a subset of the Win32 API.

WinRT APIs are built to project into several runtime langauges (C++/CX, C#,
Javascript) through interface definitions. `ros2_uwp` aims to take advantage of
the [C++/WinRT language projection](https://docs.microsoft.com/en-us/windows/uwp/cpp-and-winrt-apis/),
a native projection (using standard C++17) to consume the ROS2 `rclcpp` C++
client library.

However, there are several restrictions to UWP applications to consider. The
most relevant considerations for ROS2 are (this list is not exhaustive):
* No filesystem access, or at least not like standard Win32 apps. Impacts:
  * SROS (stores security credentials on the filesystem)
  * `ament_index`: A runtime queryable index of packages (plugin packages,
  component libraries, etc)
* No environment variables. Impacts:
  * Dynamically selecting DDS implementation
  * Various other ROS2 settings
* Restrictions on dynamic library loads. Impacts:
  * Dynamic loads must be known a-priori and built/packaged with the UWP
  application
  * Client libraries doing explict loads (`rclcpp` components, rmw_typesupport
  runtime selection) must be modified to use `LoadPackagedLibrary` instead of
  `LoadLibrary`
* Restricted API set. Impacts:
  * Win32 APIs not available to UWP applications must be avoided. Impacts:
  * All target source should be compiled targeting UWP to flag restricted APIs
  * This can be especially challenging in third party code like DDS
  implementations

... And many more considerations.

Compiling for UWP can be considered a form of cross compilation (and in fact
UWP can be targeted from `cmake` using the `CMAKE_SYSTEM_NAME` flag, commonly
used for targeting cross compilation toolchains). As with any cross compilation
there is typically one subset of code which is built for and run on the
*host* architecture, and another subset of code which is built for and run on
the *target* architecture.

ROS2 on UWP follows this paradaigm, even if the machine architecture of the
host and target are the same. The `ament` packages (the ROS2 build system) will
be compiled for and run on the host machine in a standard Win32 context. The
core ROS2 packages themselves will be built for the target architecture
targeting the UWP/WinRT runtime.


## What's Included?

This repo contains two ROS2 repos files:
* build_tools.repos - Projects used for building ROS itself, to be built and
run in a regular Win32 console context
* ros2_uwp.repos - Minimal set of ROS2 repositories to enable base
functionality up through the `rclcpp` layer (including a DDS implementation)
which have been proven to build and function within a UWP application container

## Build Instructions

### Setup
Visit Microsoft's [Installing ROS2 on Windows](https://ms-iot.github.io/ROSOnWindows/GettingStarted/SetupRos2.html). 

You'll create two build environments - one which includes ROS2 and one that only includes Visual Studio.

### Create A Workspace

Due to filepath restrictions on Windows, it's recommended to choose a short
root path:

```
mkdir c:\ros2_uwp
mkdir c:\ros2_uwp\tools
mkdir c:\ros2_uwp\target
```

### Build The Tools

Clone the build tools into the tools workspace directory. These will be needed to create a stand alone ROS install for UWP.

```
cd c:\ros2_uwp\tools
mkdir src
curl -sk https://raw.githubusercontent.com/ooeygui/ros2_uwp/master/build_tools.repos > build_tools.repos
vcs import src < build_tools.repos
```

Build the tools for the host system using `colcon`:
```
colcon build --merge-install --cmake-args -DBUILD_TESTING=OFF
```

Finally, source the tools install environment to add them to your path:
```
call c:\ros2_uwp\tools\install\local_setup.bat
```
### Build ROS2 Isolated

From the same command prompt (admin VS dev prompt with the ROS2 tools setup
script sourced), clone the `ros2_uwp` source packages
```
cd c:\ros2_uwp\target
mkdir src
curl -sk https://raw.githubusercontent.com/ooeygui/ros2_uwp/master/ros2_uwp.repos > ros2_uwp.repos
vcs import src < ros2_uwp.repos
xcopy /y src\ros2\orocos_kinematics_dynamics\orocos_kdl\config\FindEigen3.cmake src\ros2\eigen3_cmake_module\cmake\Modules
set CMAKE_PREFIX_PATH=C:\opt\rosdeps\x64\include\eigen3;%CMAKE_PREFIX_PATH%
```

### Build ROS2 for Unity Editor
The Unity Editor is a Desktop application, not a UWP application. If you would like to run ROS2 from within Unity, you'll need to build desktop binaries.

First, if you have built binaries for UWP, delete the `build` and `install` folders as we are changing the binary type.

``` batch
colcon build --merge-install --packages-ignore rmw_fastrtps_dynamic_cpp rcl_logging_log4cxx rclcpp_components ros2trace tracetools_launch tracetools_read tracetools_test tracetools_trace --cmake-args -DRMW_IMPLEMENTATION=rmw_fastrtps_cpp -DTHIRDPARTY=ON -DINSTALL_EXAMPLES=OFF -DBUILD_TESTING=OFF
```

When this is completed, you can copy the required output binaries into your unity project.

> documentation for this is included in the ros_msft_mrtk repository.

### Build ROS2 for UWP

And build ROS2 for UWP, replacing `%ROS2_ARCH%` with `Win32`, `x64`, or `ARM`

```
colcon build --parallel-workers 2 --event-handlers console_cohesion+ --merge-install --packages-ignore rcl_logging_log4cxx rcl_logging_spdlog ros2trace tracetools_launch tracetools_read tracetools_test tracetools_trace --cmake-args -A %ROS2_ARCH% -DCMAKE_SYSTEM_NAME=WindowsStore -DCMAKE_SYSTEM_VERSION="10.0" -DTHIRDPARTY=ON -DINSTALL_EXAMPLES=OFF -DBUILD_TESTING=OFF -DRCL_LOGGING_IMPLEMENTATION=rcl_logging_noop
```

### Using ROS2 in a UWP Application

High-level steps:
1. Create a c++/WinRT project in Visual Studio
2. Add c:\ros2_uwp\target\install\include to the project include path
3. Add c:\ros2_uwp\target\install\Lib to the project Lib path
4. Add necessary libraries to the project linker directives

You must include all depedent DLLs in the project itself for packaging. This is
done by right-clicking on the project and choosing 'Add existing item`. Select
all DLLs under `c:\ros2_uwp\target\install\bin. Once added, select them all,
right click, and choose 'properties'. Mark them as content to be copied and
packaged along with the application.

Finally, enable the following app capabilities in the manifest:
* `Internet (Client & Server)`
* `Private Networks`

For one example, refer to the [hololens-ros2](https://github.com/theseankelly/hololens-ros2)
project.

More standalone examples coming soon.

## Credits and Related Work

Special thanks to [Esteve Fernandez](https://github.com/esteve) who's work on
the [ros2_dotnet](https://github.com/ros2-dotnet/ros2_dotnet) project served as
a starting point for the curated repos list in this repository.

Here are some links to related work:
* [ros2_dotnet](https://github.com/ros2-dotnet/ros2_dotnet): C# language
bindings on top of the rcl layer.
* [C/C++ minimal variant](https://discourse.ros.org/t/c-c-minimal-source-tree-only-ros2-variant/11760)
: Ongoing discussions on discourse and on [REP-2001](https://github.com/ros-infrastructure/rep/pull/231)
to create a curated minimal C++ ROS2 variant. This is a similar approach but
is designed more for cross compilation to embedded targets. Uses many features
not yet available within UWP containers (class_loader, plugins, etc)
