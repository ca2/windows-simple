# windows-simple

`windows-simple` is the ca2 Windows build repository for the Simple applications. The top-level repository owns the build entry points and pins the component revisions.

| Submodule | Repository | Contents |
| --- | --- | --- |
| `operating_system/` | `ca2/windows-operating_system` | Windows platform integrations, shared MSBuild property sheets, and nested Windows and Windows-common sources |
| `port/` | `ca2/windows-port` | Third-party library ports and their nested dependencies |
| `source/` | `ca2/simple-source` | ca2 framework and Simple application sources, including nested repositories |

The Windows platform submodule pins `ca2/windows-tool` at `operating_system/operating_system-windows/tool/`. The root `tool/` directory contains the shell launcher.

## Set up the workspace

From the repository root, initialize the pinned revisions, including nested submodules.

```powershell
git submodule update --init --recursive
git submodule status --recursive
```

Check for local work and intentional revision changes before updating an existing workspace. In the status output, a leading `-` means a submodule is uninitialized, and a leading `+` means its checked-out revision differs from its parent’s pin. The relative URLs in `.gitmodules` resolve against the parent repository’s remote.

Install Visual Studio 2026 with Desktop development with C++, the MSVC `v145` toolset, and a Windows SDK. Open Developer PowerShell for Visual Studio and work from the repository root.

The current MSBuild property sheets also reference a sibling `../operating_system` workspace directory. For Debug x64, they look for ca2 headers and third-party libraries under `../operating_system/storage-windows/x64/Debug/`, as well as shared paths under `../operating_system/include/` and `../operating_system/storage-windows/x64/library/`. Provision the ca2 SDK files required by the projects at those paths.

The MSBuild pre-build events call `application_build_helper` by name. In Developer PowerShell, use `Get-Command application_build_helper` to check which copy is on `PATH`. The existing build environment uses the copy under `../operating_system/tool/bin/x64/`.

In Git Bash, run `tool/shell` from the repository root. It delegates to `operating_system/operating_system-windows/tool/shell`, which prepends the nested tool checkout’s absolute `bin/` and `bin/<architecture>/` paths to `PATH`, in that order. Use `command -v application_build_helper` to check the selected helper, and `exit` to return to the original shell.

The nested shell checks the top-level checkout’s configured `origin` URL after updating `PATH`. For an SSH origin, it sources the bundled `defer_ssh_agent` before opening the interactive Bash session. HTTPS origins skip this step. The same check applies through `shell.exe`. If an HTTPS checkout has a submodule that requires SSH, run `. "$(command -v defer_ssh_agent)"` inside the new shell to initialize the agent on demand. Use `ssh-add -l` to check the loaded identities.

## Build the Git Bash launcher

The checkout already includes a Release x64 `shell.exe`. Open it to start the installed Git Bash in this checkout and run `tool/shell`. Rebuild the launcher when its native code changes.

The standalone launcher project is under `operating_system/operating_system-windows/_tool/shell/`. Open `shell.vcxproj` in Visual Studio and select Debug or Release and a target platform. For CMake, use an x64 Developer PowerShell with CMake and Ninja on `PATH`.

```powershell
Set-Location operating_system/operating_system-windows/_tool/shell
cmake --preset x64-debug
cmake --build --preset x64-debug
```

Use the `x64-release` preset for Release. For another CMake target architecture, start a developer shell for that target and set the Ninja build directory with `-B` under `time-windows/intermediate/<architecture>/<configuration>/shell/cmake`. Pass the same configuration as `-DCMAKE_BUILD_TYPE`. The build paths use `x86`, `x64`, or `arm64`.

Both builds write `time-windows/<architecture>/<configuration>/shell.exe` and copy it to `shell.exe` at the checkout root. Use the Release x64 build when updating the checked-in executable.

## Build and verify

Use `solution-windows/simple.sln` for the main Windows build. Start with Debug x64.

```powershell
msbuild .\solution-windows\simple.sln /m /p:Configuration=Debug /p:Platform=x64
```

The solution includes the Simple Drawing launcher. After a successful build, run `time-windows/x64/Debug/_app_simple_drawing.exe` and check that its window opens and renders. The default intermediate directory is `time-windows/intermediate/`.

`solution-windows/port.sln` contains the Visual Studio projects for the ports. The root `CMakeLists.txt` is a separate entry point that delegates configuration to `source/app/_.cmake`. Its helper configuration references the ca2 tooling under `$HOME/code/operating_system/tool/`. Use an ignored out-of-source directory such as `cmake-build-windows/` for CMake. On the tested Windows setup, CMake configuration failed during Mbed TLS setup after Python 3 discovery failed, and FreeImage could not find the JPEG headers and library.

On Windows, the shared `source/app/_.cmake` setup runs `application_build_helper --defer-prepare-folder` before compiling libraries in each ca2 project directory under `source/` or `operating_system/`. It passes the absolute project source directory, MSBuild platform name, and active build configuration. The helper is normally resolved from `operating_system/operating_system-windows/tool/bin/<host architecture>/`. Set `CA2_PREPARE_FOLDER_HELPER` to override it. CMake keeps its outputs in the selected CMake build tree, with the Windows library and runtime output paths under `library/` and `output/`.

## Make changes in the owning repository

Put framework and application fixes in `source/`, platform fixes in the relevant `operating_system/` repository, port fixes in the relevant `port/` repository, and tooling fixes in `tool/`. When delivering a dependency update, record the new commit in each parent repository up to `windows-simple`. The pinned commits need to be available to collaborators who initialize the submodules. Keep generated build outputs out of commits.
