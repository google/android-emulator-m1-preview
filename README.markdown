# Note: No longer needed
Support for downloading the M1-based emulator was added to SDK Manager, so it's not necessary to go to the Github releases page to download a standalone .app anymore. In AVD Manager go to the Other Images tab as by default it doesn't show the ARM64 images.

# Android Emulator M1 Preview

This is a preview of some basic Android emulation functionality on the M1. There are still many issues, but apps work at a basic level. To be updated soon with more fixes. The release tag corresponds to this commit: <https://android.googlesource.com/platform/external/qemu/+/aca144a9e9264b11c2d729096af90d695d01455d>

## Known issues

- Webview doesn't work in the AOSP version, but works in the Google APIs version [preview v3](https://github.com/google/android-emulator-m1-preview/releases/tag/0.3). However, Chrome doesn't work.
- No device skins
- Video codecs not working
- 32 bit ARM apps won't work
- Graphical glitches in some Vulkan apps
- Popup on startup about not being able to find the ADB path (ADB will still notice the emulator if you have it installed though)
- When building, it may be faster to start then cancel the Python triggered build and then reissue `ninja -C objs install/strip` versus letting the Python triggered build finish.

## How to use

This only works on M1 Apple Silicon Macs. M1 (or equivalently capable) SoCs are required; note that this does not work on DTKs as they do not support ARM64 on ARM64 hardware virtualization via Hypevisor.framework.

Go to the Github [releases](https://github.com/google/android-emulator-m1-preview/releases) page, download a .dmg, drag to the Applications folder, and run. You'll first need to right click the app icon and select Open and then skip past the developer identity verification step (we are working on providing official identity info). The first few times it starts up it will take a while to show up, but subsequent launches will be faster.

If you've installed Android Studio and Android SDK and `adb` is available, the emulator should be visible from Studio and work (deploy built apps, debug apps, etc).

### How to configure

Edit `/Applications/Android\ Emulator.app/Contents/MacOS/aosp-master-arm64-v8a/config.ini`. Some notable options:

- `disk.dataPartition.size`: size of userdata. When reconfiguring, you'll also need to delete all `userdata*.img` files in that directory.
- `fastboot.forceColdBoot`,`fastboot.forceFastBoot`: whether to enable snapshots. Current default is snapshots disabled. Set `fastboot.forceColdBoot=no`,`fastboot.forceFastBoot=yes` to enable snapshots.
- `hw.lcd.density`: Virtual display DPI.
- `hw.lcd.width`,`hw.lcd.height`: Virtual display dimensions.
- `hw.ramSize`: RAM limit for the guest. (2GB minimum)

### How to wipe data

Remove all `userdata*.img` files in `/Applications/Android\ Emulator.app/Contents/MacOS/aosp-master-arm64-v8a/`.

## How to build your own emulator

### Building the engine

The emulator source code lives ([here](https://android.googlesource.com/platform/external/qemu/+/refs/heads/emu-master-dev)), but there are a bunch of other dependencies to download, so we use `repo`.

To build, first make sure you have Xcode and Xcode command line tools installed, and that you have Chromium `depot_tools` in your `PATH` ([link](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up)). Then:

    mkdir emu
    cd emu
    repo init -u https://android.googlesource.com/platform/external/qemu --depth=1
    repo sync -qcj 4
    cd external/qemu
    python android/build/python/cmake.py --target=darwin_aarch64

Note that canceling the python based build after it gets going and issuing just `ninja -C objs install/strip` may be faster.

The built artifacts are in `/path/to/external/qemu/objs/distribution/emulator`. They should be automatically signed. However, the binaries in `objs/` are not; to sign them, issue `./sign-objs-binaries.sh`. Note that this can only be done after `ninja -C objs install/strip` is successful.

### Building the system image

The system image is built from AOSP master `sdk_phone_arm64` with a few modifications. Ideally, let's be on a Linux host when building the system image---the build is relatively untested on M1 systems, and at least, we need to create a separate case sensitive partition for the AOSP repo. Assuming you're on Linux:

    mkdir aosp-master
    cd aosp-master
    repo init -u https://android.googlesource.com/platform/manifest -b master --depth=1
    repo sync -qcj 4

We first need to make an edit to remove all 32 bit support. Patch this change: [link](https://android-review.googlesource.com/c/platform/build/+/1518218) to `build/make/target/board/emulator_arm64/BoardConfig.mk`. Then:

    source build/envsetup.sh
    lunch sdk_phone_arm64-userdebug
    make -j12

After that's done, we can use this script to package up the system image for use in `/Applications/Android\ Emulator.app/Contents/MacOS/aosp-master-arm64-v8a/`. Assuming you're still in the Android build environment:

    echo $ANDROID_PRODUCT_OUT
    export ZIPPED_NAME=$1
    mkdir -p $ZIPPED_NAME/files
    cd $ZIPPED_NAME/files
    cp $ANDROID_PRODUCT_OUT/system-qemu.img system.img
    cp $ANDROID_PRODUCT_OUT/vendor.img vendor.img
    cp $ANDROID_PRODUCT_OUT/ramdisk.img ramdisk.img
    cp $ANDROID_PRODUCT_OUT/ramdisk.img ramdisk.img
    if [ -f $ANDROID_PRODUCT_OUT/kernel-ranchu-64 ]; then
        cp $ANDROID_PRODUCT_OUT/kernel-ranchu-64 kernel-ranchu-64
    else
        cp $ANDROID_PRODUCT_OUT/kernel-ranchu kernel-ranchu
    fi;
    cp -r $ANDROID_PRODUCT_OUT/data .
    cp -r $ANDROID_PRODUCT_OUT/advancedFeatures.ini advancedFeatures.ini
    cp -r $ANDROID_PRODUCT_OUT/userdata.img .
    cp -r $ANDROID_PRODUCT_OUT/encryptionkey.img .
    cp -r $ANDROID_PRODUCT_OUT/build.prop .
    mkdir system
    cp -r $ANDROID_PRODUCT_OUT/build.prop system/build.prop
    cp -r $ANDROID_PRODUCT_OUT/VerifiedBootParams.textproto .
    cp -r $ANDROID_PRODUCT_OUT/source.properties .

    cd ..
    zip -1rq $ZIPPED_NAME.zip files
    ls -l $ZIPPED_NAME.zip

Then, `$ZIPPED_NAME.zip` can be sent over to the M1 and the contents of its `files/` can be coped over into `/Applications/Android\ Emulator.app/Contents/MacOS/aosp-master-arm64-v8a/`.
