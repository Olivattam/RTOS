# TinyRTOS — Minimal, Modular RTOS for Small Devices

TinyRTOS is a compact, component-based real-time operating system intended for very small microcontrollers and development boards (NodeMCU/ESP8266-like devices, small Arm M0/M3 boards, and similar). Its design goal is to provide a clean, embeddable RTOS kernel with optional subsystems (networking, Bluetooth, file systems, drivers, etc.) that can be included or removed at build time so you can create minimal firmware images for resource-constrained targets.

Key ideas
- Small footprint: keep kernel and essential services minimal.
- Component-based: each feature is a self-contained component that can be toggled at build time.
- Predictable builds: configuration determines exactly which C sources and symbols are compiled.
- Pluggable board profiles: board-specific configuration files select features appropriate for the hardware.
- Seamless runtime: when a component is removed at compile time, the system still builds and runs normally — missing functionality is simply not present.

Quick highlights
- Preemptive scheduler, basic IPC (queues, semaphores), timers
- Component tree: kernel, drivers, network, storage, optional Bluetooth
- Build-time configuration via simple CMake flags or per-board config files
- Lightweight configuration scheme compatible with automated CI builds

Table of contents
- Project overview
- Component and configuration model
- Example: exclude Bluetooth for NodeMCU
- Build examples
- Runtime behavior when components are absent
- How to add a new component
- Contributing and license

Component and configuration model
- Components live under components/<component-name>/ and provide:
  - C sources
  - public headers
  - a CMake snippet or Makefile fragment that registers sources and dependency metadata
  - an optional CONFIG_<COMPONENT> Kconfig-like flag
- Configuration is resolved in layers:
  1. global defaults (configs/defaults.conf)
  2. board profile (boards/<board-name>.conf)
  3. user overrides (local build flags or a project config)
- The build system consults these flags and only includes sources for enabled components. Disabled components are not compiled and do not contribute symbols to the final image.

Example configuration flags
- CONFIG_BT (Bluetooth)
- CONFIG_WIFI
- CONFIG_NET_IPV4
- CONFIG_FS_VFAT

Example CMake-based workflow
1. Create a build directory:
   ```
   cmake -S . -B build -DPLATFORM=nodemcu -DCONFIG_BT=OFF -DCONFIG_WIFI=ON
   cmake --build build -j4
   ```
2. Alternate per-board: boards/nodemcu.conf may contain:
   ```
   CONFIG_BT=OFF
   CONFIG_WIFI=ON
   CONFIG_UART=ON
   ```

Example Makefile-based invocation
```
make PLATFORM=nodemcu CONFIG_BT=0 CONFIG_WIFI=1
```

How components are gated (example snippets)

Example Kconfig-like fragment (components/bluetooth/Kconfig)
```
config CONFIG_BT
    bool "Enable Bluetooth stack"
    default y
    depends on CONFIG_BT_HW_SUPPORT
```

Example CMakeLists snippet for a component (components/bluetooth/CMakeLists.txt)
```
if(CONFIG_BT)
    target_sources(${PROJECT_NAME} PRIVATE bt/stack.c bt/hci.c)
    target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
else()
    # Optionally compile a tiny stub providing minimal API that returns "not supported"
    # or omit entirely so no symbols exist.
endif()
```

Example: Excluding Bluetooth for NodeMCU
- NodeMCU board does not have Bluetooth hardware. In boards/nodemcu.conf:
  ```
  CONFIG_BT=OFF
  ```
- Build:
  ```
  cmake -S . -B build -DPLATFORM=nodemcu
  cmake --build build
  ```
- Result: Bluetooth component's sources are not compiled and no BT symbols are linked. The firmware image is smaller and contains only relevant drivers.

Runtime behavior when a component is absent
- If a component is compiled out, its APIs are either:
  - Unavailable at link time (encouraging compile-time checks), or
  - Provided as lightweight stubs that return "not supported" so the rest of the system stays robust.
- Recommended pattern:
  - Use compile-time guards in application code (e.g., #ifdef CONFIG_BT) to avoid calling unavailable APIs.
  - When writing libraries that may call optional functionality, check for config flags or use run-time capability checks exposed by a core "capabilities" registry.

Best practices
- Board profiles should disable all unsupported hardware components.
- App code should prefer compile-time guards for optional features to avoid surprises on tiny platforms.
- Keep drivers as small and self-contained as possible; avoid pulling large dependencies into the core kernel.

Adding a new component
1. Add a directory components/<name>/ with sources and headers.
2. Add a configuration flag (Kconfig-like) with a default.
3. Add CMake/Make fragments that gate source registration on the config flag.
4. Document dependencies and add tests or CI jobs if the component is critical.

Testing and CI
- Provide minimal test images for each supported board profile.
- Use automated builds with explicit platform flags to prevent regressions that accidentally reintroduce disabled features.

Contributing
- Open an issue to propose new features or report regressions.
- Follow the coding style in CONTRIBUTING.md (project uses small, readable C; prefer explicit guards for optional functionality).
- Include small, reproducible test cases for new drivers or subsystems.

License
- This project is released under the MIT License. See LICENSE for details.

Contact / Support
- For design discussions and hardware support requests, open issues in the repository and include the target board and build flags you used.
- Provide build logs and size comparisons if you believe a component is larger than expected.

That's it — TinyRTOS is designed to be minimal, modular, and easy to adapt to tiny boards by toggling components at build time so your final firmware contains only what the hardware supports.
