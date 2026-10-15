# panvk-faux123-driver-pixel7

Mesa **PanVK**, the open source Vulkan driver for Arm Mali GPUs, built for
Android and packaged for [AdrenoTools](https://github.com/bylaws/libadrenotools)
style driver loading on the **Google Pixel 7** (Tensor G2, Mali-G710). The
driver itself needs no root.

The driver talks to the `mali_kbase` kernel driver that already ships on the
phone, so there is no kernel to flash, and the system driver stays in place.
This build reports Vulkan 1.4 and adds features the stock driver lacks. The
version number is what the driver reports. It is not a claim that every Vulkan
1.4 feature is there: see [What works and what does not](#what-works-and-what-does-not).

I apply fixes that are not in the base I build from, and I verify every one of
them on my own hardware before I release it. Each release lists what it changes.

**This is an early, experimental driver. It is not conformant, it is faster
than the stock driver on some scenes and a little slower on others, and one
game and one Direct3D 12 test program have been run on this exact build.**

**This is a proof of concept, released as is, with no support.** See
[No support](#no-support).

**Everything here is verified on one device: my Google Pixel 7, Mali-G710 MC7,
Android 17.** It is built for that phone's kernel driver and I can make no
promises anywhere else. See [Other devices](#other-devices-not-tested-no-guarantees).

Grab the latest `.adpkg.zip` from [Releases](../../releases) and import it in an
app whose driver picker accepts a PanVK package on Mali.

## Test it on your device

- **FauxVkTest** (free, Google Play): compare this driver against your stock driver on your own device. https://play.google.com/store/apps/details?id=com.faux.vktest&referrer=utm_source%3Dgithub%26utm_campaign%3Dpanvk-faux123-driver-pixel7
- **FauxVkDB**: see how other devices score with stock and custom drivers. https://fauxvkdb.canonspike.com/

---

## Where the driver comes from

Nothing here is a repackaged binary from somewhere else. It is built from source.

| | |
|---|---|
| Source | [`gitlab.freedesktop.org/mesa/mesa`](https://gitlab.freedesktop.org/mesa/mesa), 26.3.0-devel at commit `5a07217f` |
| `mali_kbase` support | the patch series from [`zenithblue-oss/panvk-kbase-android`](https://github.com/zenithblue-oss/panvk-kbase-android), release `v0.1.0-rc3` (`fd6ed91`) |
| My patches | summarized under [What I added](#what-i-added) |
| Toolchain | Android NDK 27.3, meson cross build, `aarch64`, API level 33 |
| Packaging | `.adpkg.zip`: `libvulkan_panfrost.so` plus `meta.json` |

**Every release names the exact build**, in its release notes and in the
`meta.json` inside the package.

The build container, the patches, the test harnesses and a recreate guide live in
a private companion repository.

### Why the stock kernel driver and not Panthor

Upstream PanVK expects the open source Panthor kernel driver for this GPU. The
Pixel 7 ships Arm's `mali_kbase` kernel driver. Running PanVK on top of
`mali_kbase` means there is no kernel to flash to use it.

The patch series I build from began to recognise the Mali-G710 in its `rc3`
release, tested by its authors in a cloud lab on Android 13. My phone runs
Android 17 with a newer kernel driver. The fixes below are what it needed.

### What I added

- **The phone's whole interface restarted while a game ran.** The driver mapped
  the buffers it shares with Android's compositor in a different way than the
  compositor does. The GPU reported a fault inside the compositor, the
  compositor stopped, and Android restarted its interface. The driver now maps
  those buffers the way the stock driver does.
- The Android swapchain did not work: no frame reached the screen in an app
  that draws to an Android surface.
- Some games closed by themselves at a different point each time. The driver
  set aside room for 32 entries when an app handed it a group of textures to
  draw with, but each texture with its sampler needs two.
- The driver kept the address of one of its own helper routines in 32 bits, and
  on this kernel driver that address needs 64.
- A small leak of a file handle on every imported fence.
- Memory that an app and the GPU both use directly is taken from an uncached
  pool, so the GPU never reads a value the processor has not written out yet.
- Speed: a deferred lighting scene ran 23% slower than on the stock driver. I
  captured what both drivers send to the GPU and found two reasons. The stock
  driver hands a shader its uniform buffer values directly, where PanVK made
  the shader fetch each one from memory (52 fetches a pixel against 14). And
  when an app draws its G-buffer and its lighting in one render pass, PanVK
  switched hidden surface removal off for the G-buffer. Both are changed in
  v0.10.0. That scene is now 6% slower at 1080p and 2% slower at 2160p.
- **A game ran at under one frame a second, then lost the GPU** (Dragon Ball
  Sparking Zero in Fauxium, its opening cinematic). Two jobs on the GPU at once
  could be given the same scratch memory for the values their shaders set aside,
  so one overwrote the other's. A cloth and particle shader then read garbage
  and one pass took 5.2 seconds. Since v0.11.0 each size of scratch memory gets
  its own region, a fix carried over from my
  [GameForce Ace build](https://github.com/faux123/panvk-faux123-driver-ace).
  The cinematic now plays at 6 to 8 frames a second, and that pass takes 1.5 ms.

---

## How it was tested

Everything below was measured on **one device**: my Google Pixel 7, Android 17,
Tensor G2, Mali-G710 MC7, kernel 6.1, `mali_kbase` r54p3. The phone is rooted.
My measuring tools use root. The driver does not.

On v0.10.0:

- **Driver check** in FauxVkTest 1.11: no problems found, 28 features passed
  their check ([report 464](https://fauxvkdb.canonspike.com/report/464)).
- My feature validator, 66 checks, each run as written and deliberately broken:
  64 proven, 2 not offered. The stock driver proves 49 of them.
- **Giana Sisters: Twisted Dreams** through
  [GameNative-Mali](https://github.com/faux123/GameNative-Mali/releases)
  (Direct3D 11, DXVK `1.12-sarek`, BC texture emulation set to **None**): title,
  menus, and the start of level 1-1, with a clean picture at about 48 frames a
  second. The package was imported through the app's **Driver Manager**.
- **Direct3D 12:** the AIO Graphics Test program's Direct3D 12 scene draws at
  60 frames a second through vkd3d-proton 2.14.1. Its Vulkan, Direct3D 11, and
  Direct3D 10 scenes draw at 60 too.
- The interface restart described above happened four times in a row on an
  earlier build and has not happened since the fix. That is a handful of
  sessions, not a soak.

Frame times against the stock Mali driver, FauxVkTest speed test, in
milliseconds per frame ([report 465](https://fauxvkdb.canonspike.com/report/465)):

| scene | this driver | stock |
|---|---|---|
| `oit_linked_lists` | 19.9 | 21.1 |
| `subpasses` | 12.6 | 10.9 |
| `compute_nbody` | 15.2 | 17.3 |
| `pipeline_barriers` | 6.2 | 5.9 |
| `pipeline_barriers` at 2160p | 20.1 | 19.8 |
| `hdr` at 2160p | 62.7 | 108.6 |

So it is faster on three of these scenes, within 6% on two, and 15% slower on
one (`subpasses`).

## Other devices: not tested, no guarantees

I own one Pixel 7. Nothing here has been run on anything else. That includes
the Pixel 7 Pro, the Pixel 7a, and other Mali-G710 devices.

Concretely, on any other device:

- The driver may not load at all. Each vendor's `mali_kbase` differs, and so
  does each device's display buffer handling.
- Every fix I ship was written for the hardware I have. On a different device it
  may be unnecessary, and it may not be harmless.
- Nothing on this page was measured on that hardware, so none of the numbers
  apply to it.
- I will not look into problems on hardware I do not own.

Use it if you want to, but that is the honest state of it, and you are on your
own with it.

I build one driver per device I own, because that is the only way I can test
it: [GameForce Ace](https://github.com/faux123/panvk-faux123-driver-ace),
[Retroid Pocket 4 Pro](https://github.com/faux123/panvk-faux123-driver-rp4pro),
and [RG556](https://github.com/faux123/panvk-faux123-driver-rg556).

## Installing

**Read this first: you need an app that can load it.** Most driver pickers
were written for Adreno and refuse or ignore a Mali package.
[GameNative-Mali](https://github.com/faux123/GameNative-Mali/releases) can import
it from version 1.2.1-mali.11: open **Driver Manager**, import the zip, then
select it for a game under **Graphics**.

1. Download `panvk_faux123_pixel7_<version>.adpkg.zip` from
   [Releases](../../releases). Do not unzip it.
2. Import it in your app's driver manager, the same way you would import a Turnip
   package on an Adreno device.
3. Select it, then restart the container or game.
4. In a Windows game container, set **BCn Emulation** to **None** and use DXVK
   `1.12-sarek`. With the app's own BC texture conversion left on, the game I
   ran showed black blocks around leaves and grass.

The driver has to be loaded by an app that opts in. Android does not let you
replace the system Vulkan driver for arbitrary apps without root, so a normal
Play Store game cannot use this.

To go back, select the system driver again. Nothing on the device is replaced.

---

## What works and what does not

**If the phone's interface restarts or the phone freezes:** go back to the
system driver. If it does not respond, hold the power button to reset it.

**Games:** one Windows game was started and taken into its first level on this
build, as described above.

**Direct3D 12:** set the container's DX wrapper to VKD3D. One small test scene
was run, not a game.

**Speed:** close to the stock driver or ahead of it, as the table above shows.

**A known limit in how uniform buffers are read.** For fragment shaders the
driver copies uniform buffer values when the app records a draw. An app that
records a draw, changes the buffer afterwards, and only then submits would see
the old values. I have not met an app that does this, but I have run one game.
Setting `PANVK_UBO_PUSH=0` (Android property `debug.mesa.panvk.ubo.push`) turns
the copy off.

**Not conformant.** No conformance run was done here.

---

## No support

This is a proof of concept. I built this driver for my own device, and I am
releasing it so the community can see what is possible and build on it.

- There is no support from me. I do not answer bug reports or questions.
- I do not take requests for devices I do not own. If I do not have the device,
  I will not develop for it.
- I do not do remote debugging or beta testing.
- I may update this driver for myself from time to time and post it here. There
  is no schedule and no promise.

If you want to help, support the original developers listed under
[Credits](#credits).

---

## Credits

- **Mesa and the Panfrost team** wrote PanVK. This repository is a build of
  their work with patches on top. All the hard parts are theirs.
- **[zenithblue-oss](https://github.com/zenithblue-oss/panvk-kbase-android)**
  for the patch series that runs PanVK on `mali_kbase` on Android, which this
  build sits on.
- **[bylaws](https://github.com/bylaws/libadrenotools)** for AdrenoTools, which
  is the only reason a custom Vulkan driver can be loaded without root.
- **Khronos** for Vulkan-Samples, which turned out to be a far better driver
  test bench than any game.

## License

PanVK is [MIT licensed](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/docs/license.rst),
and so are the patches in this build. See [LICENSE](LICENSE).

The driver is compiled against Arm's `mali_kbase` interface headers, which are
GPL-2.0 with the Linux syscall note. That note is what allows a program to use
a kernel interface without taking on the kernel's license.

This project is not affiliated with or endorsed by Arm, Google, the Khronos
Group, or the Mesa project.

---

## Sponsor

These drivers are Mesa's PanVK with my patches on top; the Mesa developers did the hard part. If my builds help you, you can sponsor my work on GitHub: https://github.com/sponsors/faux123
