# OpenFOAM Development Environment on Windows

A working VS Code setup for developing, building, and debugging OpenFOAM
applications on **Windows**, where OpenFOAM itself runs inside a
**Docker container on WSL2** (tested against the ESI OpenFOAM **v2412**
build, `opencfd/openfoam-default` base image). It uses `wmake`, the
C/C++ extension's IntelliSense, and GDB.

The repository doubles as a template: `List/` is a small standalone
OpenFOAM application (`Test-List.C`, adapted from the OpenFOAM
`Test-List` example) that exercises `Foam::List`. Copy the `List/`
directory as a starting point for your own application, and reuse the
`.vscode/` configuration as-is for any new OpenFOAM app/case folder.

## Prerequisites

- **Windows 10/11** with **WSL2**, running an Ubuntu (or other
  supported) distribution, and **Docker** available inside WSL2 (Docker
  Desktop with the WSL2 backend, or Docker Engine installed directly in
  the distro).
- An **OpenFOAM container** built from an image such as
  `opencfd/openfoam-default:2412`, with OpenFOAM installed under
  `/usr/lib/openfoam/openfoam2412` inside the container (this is where
  `opencfd`'s images place it). If you use a different OpenFOAM
  version, update the version number in `.vscode/c_cpp_properties.json`
  to match.
- Inside the container/shell you build in, the OpenFOAM environment
  must be sourced (`etc/bashrc`), so that `FOAM_SRC`,
  `FOAM_USER_APPBIN`, and the `wmake`/`g++`/`gdb` toolchain are on
  `PATH`. This is normally done once in `.bashrc` when the container
  image is built.
- **Visual Studio Code** (on Windows) with:
  - [Remote - WSL](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl)
  - [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
    (to attach VS Code to the running OpenFOAM container)
  - [C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)

## Project layout

```
.
├── .vscode/
│   ├── c_cpp_properties.json   # IntelliSense include paths (container OpenFOAM install + $FOAM_SRC)
│   ├── launch.json             # GDB launch config ("OF-Debug", debugs $FOAM_USER_APPBIN/<file>)
│   ├── settings.json           # *.C/*.H as C++, OpenFOAM dict files (0/, system/, constant/, *Dict) as "OpenFOAM"
│   └── tasks.json               # "wmake-build" (default build) and "wclean" tasks
└── List/                        # Example standalone OpenFOAM application
    ├── Make/
    │   ├── files                 # Source files + EXE output name
    │   └── options                # Include paths (EXE_INC) and libs (EXE_LIBS)
    └── Test-List.C                # Application source
```

## Setup

1. Start your OpenFOAM Docker container (e.g. via `docker compose up
   -d`) and confirm you can shell into it and run `wmake -help` or
   `echo $FOAM_SRC` successfully.

2. Clone this repository **inside the WSL filesystem** (e.g. under
   `~/`), not on the Windows `/mnt/c` drive — building on `/mnt/c` is
   significantly slower and can cause path issues:

   ```bash
   git clone https://github.com/kumeric/OpenFOAM-Development-Environment-on-Windows.git
   cd OpenFOAM-Development-Environment-on-Windows
   ```

3. Open this folder with VS Code so that its editor/build/debug
   session runs **inside the OpenFOAM container**, not on the bare WSL
   host — e.g. via **Dev Containers: Attach to Running Container...**
   (pick your OpenFOAM container), or by mounting this folder into the
   container and using Remote - WSL plus `docker exec`. The
   `.vscode/c_cpp_properties.json` paths (`/usr/lib/openfoam/...`) only
   exist inside the container, so IntelliSense will show errors if VS
   Code is only attached to the WSL host.

4. Open `List/Test-List.C`.

## Build

Run the build task with **Ctrl+Shift+B** (or `Terminal > Run Task >
wmake-build`, the default build task). This runs `wmake` in the
`List/` application directory and produces the binary at
`$FOAM_USER_APPBIN/Test-List`. Use `Terminal > Run Task > wclean` to
clean build artifacts.

## Debug

Press **F5** with `Test-List.C` as the active editor tab. `launch.json`
(config name `OF-Debug`) starts GDB against
`$FOAM_USER_APPBIN/Test-List` and automatically runs the
`wmake-build` task first (`preLaunchTask`), so a fresh build always
precedes a debug session.

## IntelliSense

`c_cpp_properties.json` lists the container-side OpenFOAM `lnInclude`
directories for the libraries this template uses directly
(`OpenFOAM`, `finiteVolume`, `meshTools`, `OSspecific/POSIX`), plus
`${FOAM_SRC}/**` as a catch-all, and sets the `WM_DP` /
`WM_LABEL_SIZE=32` / `NoRepository` defines OpenFOAM itself is
compiled with (adjust `WM_LABEL_SIZE` to `64` if your build uses
64-bit labels). This requires OpenFOAM to have been built at least
once, so that the per-library `lnInclude` symlink directories exist.
`C_Cpp.intelliSenseEngine` is set to `Tag Parser` rather than the
default engine, since the full OpenFOAM header tree is large enough
that the default engine's parsing can be slow to keep up.

`settings.json` also tags common OpenFOAM case files (anything under
`0/`, `0.orig/`, `system/`, `constant/`, and any file ending in
`Dict`) with the `OpenFOAM` language mode, so case dictionaries get
appropriate syntax highlighting if you have an OpenFOAM syntax
extension installed.

## Known limitations / things to check when adapting this template

- `.vscode/c_cpp_properties.json` hardcodes the OpenFOAM version in
  its paths (`openfoam2412`). If you switch OpenFOAM versions, update
  those paths (and the `WM_LABEL_SIZE`/`WM_DP` defines if the new
  build differs).
- `List/Make/options` declares `EXE_LIBS = -lOpenFOAM`, which is
  enough for this example. If you build an application that uses
  other OpenFOAM libraries (e.g. `finiteVolume`, `meshTools`), add
  the corresponding `-I.../lnInclude` entries to `EXE_INC` and
  `-l<lib>` entries to `EXE_LIBS` — and mirror any new library paths
  in `c_cpp_properties.json` so IntelliSense picks them up too.
- `launch.json`'s `program` path resolves to
  `$FOAM_USER_APPBIN/${fileBasenameNoExtension}`, so the active editor
  tab must be the `.C` file whose compiled binary you want to debug.
- This setup assumes a single-application repository layout (one
  `Make/` directory at a fixed relative path). For a multi-application
  repo, add one `.vscode/tasks.json` entry (or a workspace per app)
  per application directory.

## License

The example application `List/Test-List.C` is adapted from OpenFOAM's
own `Test-List` tutorial source and is distributed under the terms of
the GNU General Public License v3, consistent with OpenFOAM itself.
