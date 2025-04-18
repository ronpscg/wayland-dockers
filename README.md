# Some dockers illustrating working with graphics and input

The purpose of this repo is to accompany training and perhaps some conference sessions.
It aims to show simple and important concepts that have been shown in some of the graphics training and talks, and are actually useful for day to day
work in Embedded development frameworks that use containers.

The concepts can be used in all kind of other environments (chroot + binds, cgroup/namespaces/... isolation, etc.).

Some videos explaining the why and how, including troubleshooting tips are in
- https://www.youtube.com/watch?v=Acc9mAeFjGA&list=PLBaH8x4hthVysdRTOlg2_8hL6CWCnN5l-&index=67
- https://www.youtube.com/watch?v=v8jUSUhvGwY&list=PLBaH8x4hthVysdRTOlg2_8hL6CWCnN5l-&index=68

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

