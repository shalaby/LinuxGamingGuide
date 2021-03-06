# A linux gaming guide

This is some kind of guide/compilation of things, that I got to do/learn about while on my journey of gaming on linux. I am putting it here so it can be useful to others! If you want to see something added here, or to correct something where I am wrong, you are welcome to open an issue or a PR !

## Linux distribution

I have seen many reddit posts asking which linux distributions is "best" for gaming. My thoughts on the matter is that, to get the best performance, one simply needs the latest updates. All linux distributions provide the sames packages and provide updates. Some provide them faster than others. So any distribution that updates its packages the soonest after upstream (aka the original developpers), is good in my opinion. Some distributions can take longer, sometimes 6 months after, for big projects (which is acceptable too, since one would get the updates without the initial bugs).

## Lutris

Lutris is some kind of open source Steam that helps with installing and running some games. Each game has its own install script, maintained by usually different people (as far as I understand).
I have only used Lutris, to install and run Overwatch, I don't think there's room for improvement in here since Lutris is just here to run overwatch with a chosen Wine version and environment variables. Correct me if I am wrong.

Some useful settings:
* Enable FSYNC (if you have a patched custom kernel, further information below) otherwise enable ESYNC: once overwatch is installed, go to "Configure" > "Runner Options" > Toggle FSYNC or ESYNC.

## DXVK

This is the library that maps DirectX (Windows) to Vulkan (Multi-platform and open source) so games that are meant for Windows work on Linux. It's better than wine's built-in mapper called WineD3D. Lutris provides a version already. 

You can compile your own latest one with some "better" compiler optimizations if you wish, and that's what I am doing but I have no idea about the possible FPS benefits of doing that. To do so you will need to put what DXVK's compile script gives you in `~/.local/share/lutris/runtime/dxvk/`. Link here: https://github.com/doitsujin/dxvk

```shell
git clone https://github.com/doitsujin/dxvk.git

cd dxvk

export CFLAGS="-march=native -O3 -pipe"
export CXXFLAGS="${CFLAGS}"

# Build new DLLS
./package-release.sh master ~/.local/share/lutris/runtime/dxvk/ --no-package
```
And if you feel even more adventurous, you can replace the above `CFLAGS` export with this set of flags:
```shell
export CFLAGS="-march=native -O3 -pipe -flto -floop-strip-mine -fno-semantic-interposition -fipa-pta -fdevirtualize-at-ltrans"
```
Then you go in Lutris and tell it to use this version of dxvk: "Configure" > "Runner Options" > "DXVK Version" and put `dxvk-master`

## GPU

1. Update to the latest possible driver for your GPU
2. If you are hesitating between AMD and Nvidia for your next GPU buy. As far as Linux is concerned: AMD all the way, because they are way more supported since they give out an open source driver.

## AMD GPU
A nice documentation is given by, once again, Arch's documentation: https://wiki.archlinux.org/index.php/AMDGPU

* "Very old" GPUs: the opensource driver is `radeon` and you only have that as an option, along with AMD's closed source driver I believe. But you are out of luck for running DXVK, since both driver's don't implement Vulkan.
* "Old" GPUs: GCN1 and GCN2 are now supported by the newer "amdgpu" driver and you switch to it to win a few frames.
* New GPUs: the base driver is `amdgpu`, and is shipped and updated with the linux Kernel, stacks on top of it three different drivers: 
  * Mesa: the open source graphics stack that handles AMD, Intel, Qualcomm ...etc GPUs. The AMD OpenGL driver is called RadeonSI Gallium3D and is the best you can get. The Vulkan driver is called RADV
  * amdvlk: AMD's official open source Vulkan-only driver, I suppose the rest (OpenGL) is left to mesa. link here: https://github.com/GPUOpen-Drivers/AMDVLK
  * amdgpu PRO: AMD's official closed source driver, that has its own Vulkan and OpenGL implementation. 

### Mesa / RADV

If you are running RADV and with a mesa version prior to 20.2, you should consider trying out ACO as it makes shader compilation (which happens on the CPU) way faster : go to "Configure" > "System Options" > Toggle ACO.

Your distro ships the latest stable version, you can go more bleeding edge to get the latest additions, but keep in mind that regressions often come with it. On Ubuntu there's a [PPA](https://launchpad.net/~oibaf/+archive/ubuntu/graphics-drivers) that gives out the latest mesa. Otherwise you can compile only RADV by hand with the extra bonus of using "agressive" compiler optimisations (`-march=native`, `-O3`, and LTO and PGO) and use it for any Vulkan game, in a per game basis:

```shell
git clone --depth=1 https://gitlab.freedesktop.org/mesa/mesa.git
cd mesa
mkdir build && cd build
export CFLAGS="-march=native -O3 -pipe"
export CXXFLAGS="${CFLAGS}"
meson .. \
    -D prefix="$HOME/radv-master" \
    --libdir="$HOME/radv-master/lib" \
    -D b_ndebug=true \
    -D b_lto=true \
    -D b_pgo=off \
    -D buildtype=release \
    -D platforms=drm,x11,wayland \
    -D dri-drivers= \
    -D gallium-drivers= \
    -D vulkan-drivers=amd \
    -D gles1=disabled \
    -D gles2=disabled \
    -D opengl=false
meson configure
ninja install
```
And here again, if you feel even more adventurous, you can go for this set of compiler flags:
```shell
export CFLAGS="-march=native -O3 -pipe -flto -floop-strip-mine -fno-semantic-interposition -fipa-pta -fdevirtualize-at-ltrans"
export CXXFLAGS="${CFLAGS}"
export LDFLAGS="-flto"
```

After running the lines above, you get the driver installed in `$HOME/radv-master`. Now, to use it for Overwatch (or any other game), you go to "Configure" > "System Options" > Environment variables and add the following line:

```
VK_ICD_FILENAMES=$HOME/radv/share/vulkan/icd.d/radeon_icd.x86_64.json:$OTHER_PATH/radeon_icd.i686.json
```
where you should manually replace `$HOME` by your home path `/home/Joe` and `$OTHER_PATH` by where `radeon_icd.i686.json` actually is, you can find out with
```
sudo updatedb
locate radeon_icd.i686.json
```
If the games crashes after doing all this, you can either try other git commits (you will need some git knowledge) or revert to the stable driver by simply removing the `VK_ICD_FILENAMES` environment variable. And if you don't wanna hear about bleeding edge mesa anymore you can simply remove the `mesa` folder along with `$HOME/radv-master`.

#### Profile Guided Optimisations

One can actually go even further in the compiler optimisations, by using this so called [Profile Guided Optimisations](https://en.wikipedia.org/wiki/Profile-guided_optimization). The idea behind is to produce a first version of the driver, with performance counters added in (that slow it down qute a bit). Use the driver in real life use-cases to retrieve useful statistics. Then use those statistics to compile a final, improved version, of the driver.

For what will follow, I suppose that the fist clone step has already been done, and the source code is in the `mesa` folder.

**Profiling Generation step :** 

```shell
cd path/to/mesa
git clean -f -d -x
mkdir build && cd build
export CFLAGS="-march=native -O3 -pipe -flto -floop-strip-mine -fno-semantic-interposition -fipa-pta -fdevirtualize-at-ltrans -fprofile-generate=pgo-generate"
export LDFLAGS="-flto -fprofile-generate=pgo-generate"
export CXXFLAGS="${CFLAGS}"
meson .. \
    -D prefix="$HOME/radv-master-pgogen" \
    --libdir="$HOME/radv-master-pgogen/lib" \
    -D b_ndebug=true \
    -D b_lto=true \
    -D b_pgo=generate \
    -D b_coverage=true \
    -D buildtype=release \
    -D platforms=x11,wayland \
    -D dri-drivers= \
    -D gallium-drivers= \
    -D vulkan-drivers=amd \
    -D gles1=disabled \
    -D gles2=disabled \
    -D opengl=false
meson configure
ninja install
```

This will generate the driver in the folder `$HOME/radv-master-pgogen`, what we need to do next is to tell lutris to use it by changing, in Lutris "Configure" > "System Options" > Environment variables and add/edit the following line:

```shell
VK_ICD_FILENAMES=$HOME/radv-master-pgogen/share/vulkan/icd.d/radeon_icd.x86_64.json:$OTHER_PATH/radeon_icd.i686.json
```
where you should manually replace `$HOME` by your home path `/home/Joe` and `$OTHER_PATH` by where `radeon_icd.i686.json` actually is, you can find it out by following the explanations above. After that, open and play the game for a while, let's say an hour. The game will generate some statistics files in the folder `pgo-generate` that is located at the same directory where the game's executable is located. Copy the folder to the home folder:

```shell
cp path/to/game/executable/pgo-generate ~/pgo-generate
```

**Final step: Uset hte profiling data**

We expect that profiling data is in the folder `~/pgo-generate`
```shell
export CFLAGS="-march=native -O3 -pipe -flto -floop-strip-mine -fno-semantic-interposition -fipa-pta -fdevirtualize-at-ltrans -fprofile-use=/home/Joe/pgo-generate -fprofile-partial-training"
export LDFLAGS="-flto -fprofile-use=/home/Joe/pgo-generate"
export CXXFLAGS="${CFLAGS}"
meson .. \
    -D prefix="$HOME/radv-master-pgo" \
    --libdir="$HOME/radv-master-pgo/lib" \
    -D b_ndebug=true \
    -D b_lto=true \
    -D b_pgo=use \
    -D buildtype=release \
    -D platforms=x11,wayland \
    -D dri-drivers= \
    -D gallium-drivers= \
    -D vulkan-drivers=amd \
    -D gles1=disabled \
    -D gles2=disabled \
    -D opengl=false
meson configure
ninja install
```
Where `Joe` in `-fprofile-use=/home/Joe/pgo-generate` should be changed to your username. The driver will be located in `$HOME/radv-master-pgo`, what we need to do next is to tell lutris to use it by changing, in Lutris "Configure" > "System Options" > Environment variables and add/edit the following line:

```shell
VK_ICD_FILENAMES=$HOME/radv-master-pgogen/share/vulkan/icd.d/radeon_icd.x86_64.json:$OTHER_PATH/radeon_icd.i686.json
```
where you should manually replace `$HOME` by your home path `/home/Joe` and `$OTHER_PATH` by where `radeon_icd.i686.json` actually is, you can find it out by following the explanations above.

#### Nvidia

The least one can do is redirect to Arch's documentation about it: https://wiki.archlinux.org/index.php/NVIDIA

If you didn't install the proprietary driver your computer is likely to be running an open source driver called `nouveau`, but you wouldn't want that to play games with that as it works based off reverse engineering and doesn't offer much performance.

Once you have the proprietary driver installed, open `nvidia-settings`, make sure you have set your main monitor to its maximum refresh rate and have 'Force Full Composition Pipeline' disabled (advanced settings).

Also, in Lutris, you can disable the size limit of the NVidia shader cache by adding `__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1` to the environement variables.

## Kernel

First, try to get the latest kernel your distro ships, it often comes with performance improvements (it contains the base updates for the amd gpu driver for example).

Otherwise, this is something not many touch, but using a self-compiled one can bring some small improvements. There is a git project called [linux-tkg](https://github.com/Frogging-Family/linux-tkg), which provides a script that compiles the linux Kernel from sources (takes about ~30mins) with some customization options : the default [scheduler](https://en.wikipedia.org/wiki/Scheduling_(computing)) ([CFS](https://en.wikipedia.org/wiki/Completely_Fair_Scheduler)) can be changed to other ones (Project C UPDS, PDS, BMQ, MuQSS) and can recieve the so called ["FSYNC"](https://steamcommunity.com/games/221410/announcements/detail/2957094910196249305) patch. Both of these can help getting better performance in games. And also other patches. Linux-tkg needs to be compiled on your own machine (where you can use compiler optimisations such as `-O3` and `-march=native`) with an interactive script and a config file, I worked on the script to install on Ubuntu and Fedora. link here: https://github.com/Frogging-Family/linux-tkg

For a less efforts solution, you can look up Xanmod kernel, Liquorix, Linux-zen. That provide precompiled binaries.

## Game mode

It's a small program that puts your computer in performance mode when you start your game, it's available in most distro's repositories and I believe it helps in giving consistent FPS. Lutris uses it automatically if it's detected, otherwise you need to go, for Overwatch, "Configure" > "System Options" > "Environment variables" and add `LD_PRELOAD="$GAMEMODE_PATH/libgamemodeauto.so.0"` where you should replace `$GAMEMODE_PATH` with the actual path (you can do a `locate libgamemodeauto.so.0` on your terminal to find it). Link here: https://github.com/FeralInteractive/gamemode.

You can check whether or not gamemode is running with the command `gamemoded -s`. For GNOME users, there's a status indicator shell extension that show a notification and a tray icon when gamemode is running: https://extensions.gnome.org/extension/1852/gamemode/

## Wine

Wine can have quite the impact on Overwatch, both positive and negative. Latest wine from Lutris works fine. You can give a try to wine-tkg here, it offers quite the amount of performance patches and, for overwatch, improve the game's performance. It also can be compiled with more agressive optimisations, though usually they brake the games... Link here : 
https://github.com/Frogging-Family/wine-tkg-git


## X11/Wayland

I use only X11 for now, and works nicely. Wayland is not as good as X11 for now, for gaming (the desktop smoothness is better than x11 though, with AMD GPUs), except if you want to try a custom wine with Wayland patches: https://github.com/varmd/wine-wayland. I am unable to run Overwatch with it yet.

## Performance overlays

Two possibilities:

* MangoHud: It is available in the repositories of most linux distros, to activate it, you only need to add the environment variable `MANGOHUD=1`, the stats you want to see in `MANGOHUD_CONFIG`. More information here: https://github.com/flightlessmango/MangoHud
* DXVK has its own HUD and can be enabled by setting the variable `DXVK_HUD`, the possible values are explained in [its repository](https://github.com/doitsujin/dxvk)

## OBS/Streaming

Works nicely with X11 on AMD GPUs, actually better than Windows, since you can use VAAPI-FFMPEG on Linux, and it has a better video quality than the AMD thingy on windows. Nvidia has been reported to work nicely on linux and on windows with their new NVENC thing.

On Gnome, an experimental feature can be enabled: 
```shell
gsettings set org.gnome.mutter experimental-features '["dma-buf-screen-sharing"]'
```
That will enable `zero-copy` game capture, which noticeably reduces the added input lag added by the game capture for streaming. A beta version of `Obs-studio` should then be used, it can be installed via `flatpak`
```shell
flatpak install --user https://flathub.org/beta-repo/appstream/com.obsproject.Studio.flatpakref
```
Then it can be run with an environment variable, `OBS_USE_EGL=1`:
```shell
OBS_USE_EGL=1 com.obsproject.Studio
```
where `com.obsproject.Studio` is the name of the `obs-studio` executable, installed through flatpak, it may have another name in your specific distro.

## Compositor / desktop effects

The compositor is the part of your DE that adds desktop transparency effects and animations. In games, this can result in a noticeable loss in fps and added input lag. Some DEs properly detect the fullscreen application and disable compositing for that window, others don't. Gnome, if recent enough, disables the compisitor for fullscreen apps. Luckily, apparently, Lutris has a system option called Disable desktop effects which will disable compositing when you launch the game and restore it when you close it.

#### Misc
* Background YT videos: If you have youtube music in the background, try to switch to an empty tab and not leave the tab on the video. I noticed that like this the video doesn't get rendered and helps freeing your GPU or CPU (depending on who is doing the decoding).
* KDE file indexer : If you're using KDE, you may consider disabling the file indexer. This is either done in the KDE settings or with balooctl disable (requires a reboot).
