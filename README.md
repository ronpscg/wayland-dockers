# Some dockers illustrating working with graphics and input

The purpose of this repo is to accompany training and perhaps some conference sessions.
It aims to show simple and important concepts that have been shown in some of the graphics training and talks, and are actually useful for day to day
work in Embedded development frameworks that use containers.

The concepts can be used in all kind of other environments (chroot + binds, cgroup/namespaces/... isolation, etc.).

Some videos explaining the why and how, including troubleshooting tips are in:
- [Using Wayland and GPU acceleration inside Docker (Host: Ubuntu 24.10, Docker: Ubuntu 25.04)](https://www.youtube.com/watch?v=Acc9mAeFjGA&list=PLBaH8x4hthVysdRTOlg2_8hL6CWCnN5l-&index=67)
- [Using electronjs and electronforge inside a docker with wayland](https://www.youtube.com/watch?v=v8jUSUhvGwY&list=PLBaH8x4hthVysdRTOlg2_8hL6CWCnN5l-&index=68)

## Building
You can of course select your own names, tags, etc.

```
docker build -t wayland-client-glmark2-tests -f Dockerfile.wayland-client-glmark2-tests .
```

## Wayland Client tests
**Disclaimer**: Some architectures (e.g. NXP iMX8) require additional device driver exposure, and not just what is in /dev/dri/... . If you are on such an architecture,
you will not get your graphic acceleration.

**Note**: This addresses *only* wayland, and does not count on DRMFB so if you expect Xwayland and using /dev/fb0... DISPLAY... etc. - they will not be here as they are not necessary.


Fine grained providing the /dev/dri/* devices

```
docker run --rm -it -device /dev/dri -v $XDG_RUNTIME_DIR:$XDG_RUNTIME_DIR -v /run/user/$UID/$WAYLAND_DISPLAY:/run/user/$UID/$WAYLAND_DISPLAY -e WAYLAND_DISPLAY=$WAYLAND_DISPLAY -e XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR wayland-client-glmark2-tests:latest
```



Fine grained, indicating cgroup, and bind mounting /dev/dri
```
docker run --rm -it -v /dev/dri:/dev/dri --device-cgroup-rule='c 226:* rmw' -v $XDG_RUNTIME_DIR:$XDG_RUNTIME_DIR -v /run/user/$UID/$WAYLAND_DISPLAY:/run/user/$UID/$WAYLAND_DISPLAY -e WAYLAND_DISPLAY=$WAYLAND_DISPLAY -e XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR wayland-client-glmark2-tests:latest
```

One could instead of providing /dev/drm/ ... etc run the container as --privileged which is of course less secure, but this repo is not a book, and if I use it in a talk you will hear about that enough...


### Checking that graphic acceleration works
You should see something that is **not** llvmpipe  in the GL_RENDERER of the output.
e.g.:

Graphic accelerated:
```
=======================================================
    glmark2 2023.01
=======================================================
    OpenGL Information
    GL_VENDOR:      Intel
    GL_RENDERER:    Mesa Intel(R) Iris(R) Xe Graphics (ADL GT2)
    GL_VERSION:     OpenGL ES 3.2 Mesa 25.0.3-1ubuntu2
    Surface Config: buf=32 r=8 g=8 b=8 a=8 depth=24 stencil=0 samples=0
    Surface Size:   800x600 windowed
=======================================================

```

**Not** graphic accelerated
```
...
    GL_VENDOR:      Mesa
    GL_RENDERER:    llvmpipe (LLVM 19.1.7, 256 bits)
...
```


## Electron.js example building on the wayland client tests

Building:
```
docker build -t wayland-client-electronjs-app -f Dockerfile.wayland-client-electronjs-app .
```

Running:
```
docker run --name electron --rm -it -v /dev/dri:/dev/dri --device-cgroup-rule='c 226:* rmw' -v $XDG_RUNTIME_DIR:$XDG_RUNTIME_DIR -v /run/user/$UID/$WAYLAND_DISPLAY:/run/user/$UID/$WAYLAND_DISPLAY -e WAYLAND_DISPLAY=$WAYLAND_DISPLAY -e XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR wayland-client-electronjs-app
```

## Tauri 2.0 example building on the wayland client tests
There are multiple options and technologies to build Tauri, and one is selected, arbitrarily as an example.
The package manager dependencies are not as one would assume, and using `npm` would require installing `pnpm` this way or another.
Since I did not want (at this point) to waste time exploring too many of these things, I just installed the same packages from the Electron docker, but without building the electron app itself, and that's it.

Do note that the selection to run the `--debug` build is arbitrary, and is meant only to show hot reloading etc. Building it takes **a lot** of time (could easily be close to 10 minutes), so one is selected in the Dockerfile, and the Dockerfile itself contains quite a lot of comments.


Building:
```
docker build -t wayland-client-tauri-app -f Dockerfile.wayland-client-tauri-app .
```
If you find yourself wondering about why *~/.profile* is sourced, the reason is that the `cargo`, `rustc`, `rustup`, ... executables are
in a path that need to be included, and as by default a *non-login* shell is run, they need to be introduced. You can alternatively run
`bash -ci` before the commands you run (not only in building, also in general, e.g. in a `docker run` statement), to have *$HOME/.profile* "automatically" sourced for you, and therfore have the Rust command environment properly set.

Running:
```
docker run --name tauri --rm -it -v /dev/dri:/dev/dri --device-cgroup-rule='c 226:* rmw' -v $XDG_RUNTIME_DIR:$XDG_RUNTIME_DIR -v /run/user/$UID/$WAYLAND_DISPLAY:/run/user/$UID/$WAYLAND_DISPLAY -e WAYLAND_DISPLAY=$WAYLAND_DISPLAY -e XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR wayland-client-tauri-app:latest /tauri/my-app/src-tauri/target/debug/my-app
```

Running and showing hot reload (Warning: even if there are no changes at all, npm run tauri dev can take almost a minute, which is anonying)
```
docker run --name tauri --rm -it -v /dev/dri:/dev/dri --device-cgroup-rule='c 226:* rmw' -v $XDG_RUNTIME_DIR:$XDG_RUNTIME_DIR -v /run/user/$UID/$WAYLAND_DISPLAY:/run/user/$UID/$WAYLAND_DISPLAY -e WAYLAND_DISPLAY=$WAYLAND_DISPLAY -e XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR wayland-client-tauri-app:latest bash

# Then inside the docker:
npm run tauri dev
```

One can then change a file and see the hot reload
Note: known issues with Tauri as per the time of writing it (April 19, 2025), may require you to reload the page on the frontend window on the second time you run this command

### Cross-Compiling note:
When cross-compiling an application for another architectures, it is always better to either
- Use cross-compilers
- Build on a native architecture host machine. For example: 
  - There are now arm64 servers in most of the cloud providers.
  - You can see notes about building for arm64 Linux machines on Apple Silicon on a video I posted on Youtube called [Linux on Macbooks - in and out of MacOS](https://www.youtube.com/watch?v=j5ajUgxmqKU&list=PLBaH8x4hthVysdRTOlg2_8hL6CWCnN5l-&index=31), and understand the different trade-offs and best building speed alternative (e.g. dockers/containers, emulators (hypervisor framework) and running Linux directly on your Mac device).

Unfortunately, **cross-compiling in Tauri for different architectures is fundamendtally broken**. It might be possible to copy your own *sysroots* to a host and fight with it, but I did not have the time to do so. Surprisingly, the `cross` crate also fails miserably solving this, and the only recommended ways, actually also by Tauri upstream are to (assume an x86_64 host and an aarch64 target):
- Use `--platform linux/arm64` and suffer the consequences (slowness)
- Use a native machine, and or a native runner should you engage in a github related flow

Given these reasons, a couple of ways of demonstrating a Tuari split are presented, and to allow for showing a CI/CD pipeline, while not conteminating this repo (which is beautifully built, but once I add a github workflow it won't be so beautiful ;-) ), I may suggest a split.
Cross compiling other `rust` apps is very well explained in https://github.com/ronpscg/fbdev-evdev-simple-test-apps .

