# Burrito 🌯
## Cross-Platform Elixir Deployments

* [What Is It?](#what-is-it)
  * [Background](#background)
  * [Feature Overview](#feature-overview)
  * [Technical Component Overview](#technical-component-overview)
  * [End-To-End Overview](#end-to-end-overview)
* [Quick Start](#quick-start)
  * [Experimental Disclaimer](#disclaimer)
  * [Preparation and Requirements](#preparation-and-requirements)
  * [Mix Project Setup](#mix-project-setup)
  * [Mix Release Config Options](#mix-release-config-options)
  * [Build-Time Environment Variables](#build-time-environment-variables)
  * [Application Entry Point](#application-entry-point)
  * [Maintenance Commands](#maintenance-commands)
* [Advanced Build Configuration](#advanced-build-configuration)
  * [Build Steps and Phases](#build-steps-and-phases)
  * [Build Targets and Qualifiers](#build-targets-and-qualifiers)
  * [Using custom ERTS builds](#using-custom-erts-builds)
* [Known Limitations and Issues](#known-limitations-and-issues)
  * [Runtime Requirements](#runtime-requirements)
* [Contributing](#contributing)
  * [Welcome!](#welcome)

## What Is It?

#### Background
Burrito is our answer to the problem of distributing Elixir CLI applications across varied environments, where we cannot guarantee that the Erlang runtime is installed, and where we lack the permissions to install it ourselves. In particular, we have CLI tooling that must be deployed on-premise, by consultants, into customer environments that may be running MacOS, Linux, or Windows.

Furthermore, these tools depend on NIFs that we need to cross-compile for any of the environments that we support, from one common build server, running in our CI environment.

We were heavily inspired by [Bakeware](https://github.com/bake-bake-bake/bakeware), which lays a lot of the ground work for our approach. Ultimately we implemented and expanded upon many of Bakeware's ideas using [Zig](https://ziglang.org/).

#### Feature Overview
* Builds a self-extracting archive for a Mix project, targeting Windows, MacOS, and Linux, containing:
  * Your compiled BEAM code
  * The required ERTS for your project
  * Compilation artifacts for any [elixir-make](https://github.com/elixir-lang/elixir_make) based NIFs used by the project
* Provides a "plugin" interface for injecting Zig code into your application's boot sequence
  * We use this to perform automatic updates and licensing checks (see `lib/versions/release_file.ex` for details)
* Automatically uninstalls old versions of the payload if a new version is run.

#### Technical Component Overview
Burrito is composed of a few different components:
* **Mix Release Module** - A module that is executed as a Mix release step. This module takes care of packing up the files, downloading and copying in different ERTS runtimes, and launching the Zig Archiver and Wrapper.
* **Zig Archiver** - A small Zig library that packs up an entire directory into a tar-like blob. This is known as the "payload" -- which will contain all the compiled BEAM code for your release, and the ERTS for the target platform. This is Gzip compressed and then embedded directly into the wrapper program.
* **Zig Wrapper** - This is portable cross-platform Zig code that wraps around the payload generated during the Mix release process.

```
      Burrito Produced Binary
┌────────────────────────────────┐
│                                │
│       Zig Wrapper Binary       │ <---- Compiled from `wrapper.zig`
│                                │
├────────────────────────────────┤
│        Payload Archive         │
│ ┌────────────────────────────┐ │
│ │                            │ │
│ │    ERTS Native Binaries    │ <------ If cross-compiling, this is downloaded from a build server
│ │                            │ │
│ └────────────────────────────┘ │
│                                │ <---- This bottom payload portion is generated by `archiver.zig`
│ ┌────────────────────────────┐ │
│ │                            │ │
│ │   Application BEAM Code    │ │
│ │                            │ │
│ └────────────────────────────┘ │
│                                │
└────────────────────────────────┘
```

#### End To End Overview

 1. You build a Burrito wrapped binary of your application and send it to an end-user
 2. The end-user launches your binary like any other native application on their system
 3. In the background (first-run only) the payload is extracted into a well defined location on the system. (AppData, Application Support, etc.)
 4. The wrapper executes the Erlang runtime in the background, and transparently launches your application within the same process
 5. Subsequent runs of the same version of that application will use the previously extracted payload

## Quick Start
#### Disclaimer
Burrito was built with our specific use case in mind, and while we've found success with deploying applications packaged using Burrito to a number of production environments, the approach we're taking is still experimental.

That being said, we're excited by our early use of the tooling, and are eager to accept community contributions that improve the reliability of Burrito, or that add support for additional platforms.

#### Preparation and Requirements

**NOTE:** Due to current limitations of Zig, some platforms are less suited as build machines than others: we've found the most success building from Linux and MacOS. The matrix below outlines which build targets are currently supported by each host.

|Target| Host | Host | Host | Host |
|--|--|--|--|--|
| | Windows x64 | Linux | MacOS (x86_64) | MacOS (Apple Silicon)** |
| Windows x64 |❌|✅|✅|❌|
| Linux |❌|✅|✅|❌|
| MacOS (x86_64) |❌|⚠️*|✅|❌|
| MacOS (Apple Silicon)** |❌|⚠️*|✅|✅|

\* NIFs implemented using `elixir-make` cannot be cross-compiled from Linux to MacOS, pending a [proposed linker change in Zig](https://github.com/ziglang/zig/issues/8180)

** Automated testing and building of Apple silicon Erlang builds is blocked on [support for Apple Silicon in Github Actions](https://github.com/actions/virtual-environments/issues/2187).

----

You must have the following installed and in your PATH:

* Zig (0.9.0) -- `zig`
* XZ -- `xz`
* 7z -- `7z` (For Windows Targets)

----

#### Mix Project Setup

1. Add `burrito` to your list of dependencies:

```elixir
defp deps() do
  [{:burrito, github: "burrito-elixir/burrito"}]
end
```

2. Create a `releases` function in your `mix.exs`, add and configure the following for your project:

```elixir
  def releases do
  [
    example_cli_app: [
      steps: [:assemble, &Burrito.wrap/1],
      burrito: [
        targets: [
          macos: [os: :darwin, cpu: :x86_64],
          linux: [os: :linux, cpu: :x86_64],
          windows: [os: :windows, cpu: :x86_64]
        ],
      ]
    ]
  ]
  end
```

(See the [Mix Release Config Options](#mix-release-config-options) for additional options)

3. To build a release for all the targets defined in your `mix.exs` file: `MIX_ENV=prod mix release`
4. You can also build a single target by setting the `BURRITO_TARGET` environment variable to the alias for that target (e.g. Setting `BURRITO_TARGET=macos` builds only the `macos` target defined above.)

NOTE: In order to speed up iteration times during development, if the Mix environment is not set to `prod`, the binary will always extract its payload, even if that version of the application has already been unpacked on the target machine.

#### Mix Release Config Options

* `targets` - A list of atoms, the targets you want to build for (`:darwin`, `:win64`, `:linux`, `:linux_musl`) whenever you run a `mix release` command -- if not defined, defaults to native host platform only.
* `debug` - Boolean, will produce a debug build if set to true. (Default: `false`)
* `no_clean` - Boolean, will not clean up after building if set to true. (Default: `false`)
* `plugin` - String, a path to a Zig file that contains a function `burrito_plugin_entry()` which will be called before unpacking the payload at runtime. See [the example application for details.](example/test_plugin/plugin.zig)

#### Build-Time Environment Variables

* `BURRITO_TARGET` - Override the list of targets provided in your release configuration. (ex: `BURRITO_TARGET=win64`, `BURRITO_TARGET=linux,darwin`)

#### Application Entry Point
For Burrito to work properly you must define a `:mod` in your project's Mix config:
```elixir
  def application do
    [
      mod: {MyEntryModule, []}
    ]
  end
```
This module must implement the callbacks defined by the [`Application`](https://hexdocs.pm/elixir/1.12/Application.html) module, as stated in the Mix documentation:

```elixir
defmodule MyEntryModule do
  def start(_, _) do
   # Returning `{:ok, pid}` will prevent the application from halting.
   # Use System.halt(exit_code) to terminate the VM when required
  end
end

```

If you wish you retrieve the argv passed to your program by Burrito use this snippet:
```elixir
 args = Burrito.Util.Args.get_arguments() # this returns a list of strings
 ```

#### Maintenance Commands
Binaries built by Burrito include a built-in set of commands for performing maintenance operations against the included application:

* `./my-binary maintenance uninstall` - Will prompt to uninstall the unpacked payload on the host machine.

## Advanced Build Configuration

#### Build Steps and Phases

Burrito runs the mix release task in three "Phases". Each of these phases contains a number of "Steps", and a context struct containing the current state of the build, which is passed between each step.

The three phases of the Burrito build pipeline are:

  * `Fetch` - This phase is responsible for downloading or copying in any replacement ERTS builds for cross-build targets.
  * `Patch` - The patch phase injects custom scripts into the build directory, this phase is also where any custom files should be copied into the build directory before being archived.
  * `Build` -  This is the final phase in the build flow, it produces the final wrapper binary with a payload embedded inside.

  You can add your own steps before and after phases execute. Your custom steps will also receive the build context struct, and can return a modified one to customize a build to your liking.

  An example of adding a step before the fetch phase, and after the build phase:

  ```elixir
  # ... mix.exs file
  def releases do
    [
      my_app: [
        steps: [:assemble, &Burrito.wrap/1],
        burrito: [
          # ... other Burrito configuration
          extra_steps: [
            fetch: [pre: [MyCustomStepModule, AnotherCustomStepModule]],
            build: [post: [CustomStepAgain, YetAnotherCustomStepModule]]
          ]
        ]
      ]
    ]
  end
  ```

  You can override default steps or phases by setting the `phases` option.

  If your build phase's requirements are different to Burrito's, specify your own `assemble` step that calls `Burrito.Builder.build(release)`.

  ```elixir
    # ... mix.exs file
    def releases do
      [
        my_app: [
          steps: [:assemble, &MyProject.wrap/1],
          burrito: [
            # ... other Burrito configuration
            phases: [
              build: [MyProject.CustomBuildStep1, MyProject.CustomBuildStep2]
            ]
          ]
        ]
      ]
    end
  ```

  ```elixir
  defmodule MyProject
    def wrap(%Mix.Release{} = release) do
      pre_check(release)
      Burrito.Builder.build(release)
    end

    defp pre_check(release) do
      # checks specific to your build.
    end
  end
  ```

#### Build Targets and Qualifiers

A Burrito build target is a keyword list that contains an operating system, a CPU architecture, and extra build options (called Qualifiers).

Here's a definition for a build target configured for Linux x86-64:

```elixir
targets: [
  linux: [os: :linux, cpu: :x86_64]
]
```

Build targets can be further customized using build qualifiers. For example, a Linux build target can be configured to use `musl` instead of `glibc` using the following definition:

```elixir
targets: [
  linux_musl: [
    os: :linux,
    cpu: :x86_64,
    libc: :musl
  ]
]
```

Build qualifiers are a simple way to pass specific flags into the Burrito build pipeline. Currently, only the `libc` and `custom_erts` qualifiers have any affect on the standard Burrito build phases and steps.

Tip: You can use these qualifiers as a way to pass per-target information into your custom build steps.

#### Using Custom ERTS Builds

The Burrito project provides precompiled builds of Erlang for the following platforms:

```elixir
[os: :darwin, cpu: :x86_64],
[os: :linux, cpu: :x86_64, libc: :glibc], # or just [os: :linux, cpu: :x86_64]
[os: :linux, cpu: :x86_64, libc: :musl],
[os: :windows, cpu: :x86_64]
```

If you require a custom build of ERTS, you're able to override the precompiled binaries on a per target basis by setting `custom_erts` to the path of your ERTS build:

```elixir
targets: [
  linux_arm: [
    os: :linux,
    cpu: :arm64,
    custom_erts: "/path/to/my_custom_erts.tar.gz"
  ]
]
```

The `custom_erts` value should be a path to a local `.tar.gz` of a release from the Erlang source tree. The structure inside the archive should mirror:

```
. (TAR Root)
└─ otp-A.B.C-OS-ARCH
  ├─ erts-X.Y.Z/
  ├─ releases/
  ├─ lib/
  ├─ misc/
  ├─ usr/
  └─ Install
```

You can easily build an archive like this by doing the following commands inside the (official Erlang source code)[https://github.com/erlang/otp]:

```bash
# configure and build Erlang as you require...
# ...

export RELEASE_ROOT=$(pwd)/release/otp-A.B.C-OS-ARCH
make release
cd release
tar czf my_custom_erts.tar.gz otp-A.B.C-OS-ARCH
```

## Known Limitations and Issues
#### Runtime Requirements
Minimizing the runtime dependencies of the package binaries is an explicit design goal, and the requirements for each platform are as follows:
##### Windows
* MSVC Runtime for the Erlang version you are shipping
* Windows 10 Build 1511 or later (for ANSI color support)
##### Linux
* Any distribution with glibc (or musl libc)
* libncurses-5
##### MacOS
* No runtime dependencies, however a security exemption must be set in MacOS Gatekeeper unless the binary undergoes codesigning

## Contributing
#### Welcome!
We are happy to review and accept pull requests to improve Burrito, and ask that you follow the established code formatting present in the repo!

Everything in this repo is licensed under The MIT License, see `LICENSE` for the full license text.
