# Building braw-decode on macOS

## Prerequisites

Download the Blackmagic RAW SDK for macOS from:
https://www.blackmagicdesign.com/developer/products/braw/sdk-and-software

A free Blackmagic Design account is required. The downloaded file will be
`Blackmagic_RAW_Macintosh_<version>.zip`.

## Extracting the SDK

The zip contains a `.dmg`. Mount it and expand the `.pkg` installer without
running it:

```sh
unzip Blackmagic_RAW_Macintosh_5.1.zip
hdiutil attach Blackmagic_RAW_5.1.dmg -mountpoint /tmp/brawsdk_mounted
pkgutil --expand "/tmp/brawsdk_mounted/Install Blackmagic RAW 5.1.pkg" /tmp/brawsdk_pkg
mkdir /tmp/brawsdk_extracted
cd /tmp/brawsdk_extracted
cpio -idmv < /tmp/brawsdk_pkg/BlackmagicRawSDK.pkg/Payload
```

The SDK files will be at:
`/tmp/brawsdk_extracted/Applications/Blackmagic RAW/Blackmagic RAW SDK/Mac/`

Copy the `Include` and `Libraries` directories into the project root:

```sh
cp -r ".../Mac/Include" /path/to/braw-decode/
cp -r ".../Mac/Libraries" /path/to/braw-decode/
```

The `Libraries` directory contains `BlackmagicRawAPI.framework`, a universal
binary supporting both x86_64 and arm64.

## Code changes required for macOS

The original project was written for Linux. The Mac SDK has a different API
surface in several places.

### Makefile

Replace `-ldl` with `-framework CoreFoundation`. The detection is done with
`uname -s` so the Makefile remains compatible with Linux.

### src/braw.h

- Include `<CoreFoundation/CoreFoundation.h>` on Mac.
- The `lib` field (factory load path) must be `CFStringRef` on Mac instead of
  `const char*`. Use `CFSTR("Libraries")`.
- The `SidecarMetadataParseWarning` and `SidecarMetadataParseError` callback
  signatures use `CFStringRef` parameters on Mac instead of `const char*`.
- Change `frameCount` in `BrawInfo` from `long unsigned int` to `uint64_t` to
  match the Mac SDK's `GetFrameCount` signature.

### src/braw.cpp

- `IBlackmagicRaw::OpenClip` takes a `CFStringRef` on Mac instead of
  `const char*`. Convert the filename with
  `CFStringCreateWithCString(nullptr, filename.c_str(), kCFStringEncodingUTF8)`
  and release it after the call.
- Guard `codec->FlushJobs()` in the destructor with a null check. When no file
  is provided the codec is never initialised, causing a segfault on exit.

## Building

```sh
make
```

The binary is placed at `braw-decode` inside the project directory. It must be
run from there so it can locate `Libraries/BlackmagicRawAPI.framework` via the
relative path.
