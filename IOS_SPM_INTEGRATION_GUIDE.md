# Travel SDK — iOS Swift Package Manager Build & Integration Guide

Complete process for shipping the Travel SDK Flutter module to iOS as a
Swift Package — no CocoaPods anywhere in the consuming app. Part A is "how
to build the package from this Flutter module" (do this once, and again on
every version bump). Part B is "how an integrating app adds the finished
package." Part C is troubleshooting.

This guide is self-contained for handover purposes. The narrower, app-facing
version of Part B also exists inline in `NATIVE_INTEGRATION_GUIDE.md` (in
the SDK repo) for teams who only need the integration step.

## Download the prebuilt package

**[Download `FlutterSDKPackage`](https://drive.google.com/drive/folders/1fr6UXBntGd7DmD7ll4kpkQKKnz5DeZli?usp=sharing)**
— a ready-to-use, self-contained Swift Package (already built per Part A
below). Unzip it and follow **Part B** to integrate it into your iOS app —
you don't need to do the Part A build steps yourself unless you're handling
a version bump.

---

## How this fits together

```
Flutter module (Dart source)
  │  flutter build ios-framework
  ▼
Flutter.framework + App.xcframework + one xcframework per plugin
  │  (some plugins' real native SDK isn't vendored by Flutter — see Part A.4)
  ▼
FlutterSDKPackage/  (a Swift Package: Package.swift + binaryTargets)
  │  Add Local Package / Add Package Dependency
  ▼
Host iOS app — `import FlutterSDK`, no CocoaPods involved
```

The hard part, and the bulk of this guide, is Part A.4: Flutter's own build
vendors most plugins as working xcframeworks, but a handful of third-party
SDKs this module depends on (Firebase's binary-only pieces, Google Maps,
Adjust, and a cluster of Firebase-internal libraries) are either not
vendored at all, or vendored in a form Swift Package Manager can't embed
correctly. Every one of those was discovered by trial and error — building,
running, reading the exact `dyld` crash, and fixing that one thing — and
this guide gives you the same recipes plus the diagnostic method, so a
future version bump (new plugin, new SDK version) doesn't mean
rediscovering them from scratch.

---

# Part A — Building the SPM package from a Flutter module

## A.0 Prerequisites

- The Flutter module's source (a `pubspec.yaml`-rooted Flutter project, not
  a Flutter *app* — this is the add-to-app workflow).
- Xcode 15+ and the Flutter SDK installed and on `PATH`.
- CocoaPods installed (`pod --version`) — used only as a **local build
  tool** during Part A.4, never required by the integrating app.
- macOS, since all of this is Xcode/xcodebuild-driven.

## A.1 Build the Flutter xcframeworks

From the Flutter module's root:

```sh
flutter build ios-framework --xcframework --no-debug --no-profile --release \
  --obfuscate --split-debug-info=./debug_info
```

This produces `build/ios/framework/Release/` containing `Flutter.xcframework`,
`App.xcframework`, `FlutterPluginRegistrant.xcframework`, and one
xcframework per Flutter plugin the module depends on (and, transitively, an
xcframework for some — not all — of the native SDKs those plugins wrap).
`--obfuscate`/`--split-debug-info` are optional hardening flags; drop them
for faster iteration while you're still working through Part A.4.

Copy (or symlink) that `Release/` output into wherever your SPM package will
live, under a folder named `Frameworks/`. In this project that's
`native_example/TravelSDKiOSDemo/FlutterSDK/Frameworks/` (in the SDK repo),
shared with the legacy CocoaPods packaging; for a fresh module you can just
put it directly inside the new SPM package folder.

## A.2 Create the SPM package skeleton

```
FlutterSDKPackage/
├── Package.swift                  (generated — see A.3, never hand-edit)
├── generate_package_swift.sh       (the generator — copy from this repo)
├── Frameworks/                     (symlink or copy of build/ios/framework/Release)
├── VendoredThirdParty/             (xcframeworks you'll add in A.4 — empty for now)
└── Sources/FlutterSDK/
    └── FlutterSDK.swift            (empty placeholder — SPM requires ≥1 source file)
```

```sh
mkdir -p FlutterSDKPackage/Sources/FlutterSDK FlutterSDKPackage/VendoredThirdParty
echo '// Umbrella module that links every vendored xcframework binaryTarget together.' \
  > FlutterSDKPackage/Sources/FlutterSDK/FlutterSDK.swift
ln -s ../path/to/build/ios/framework/Release FlutterSDKPackage/Frameworks
```

## A.3 Generate `Package.swift`

This is the actual, working generator script for this module — already in
the repo at
`native_example/TravelSDKiOSDemo/FlutterSDKPackage/generate_package_swift.sh`,
reproduced here for reference. It globs every `.xcframework` in
`Frameworks/` and `VendoredThirdParty/` and emits one `binaryTarget` per
bundle, plus an umbrella `FlutterSDK` target that depends on all of them —
so the target list never needs hand-maintaining as plugins are added or
removed across Flutter releases. The `collides_with_firebase_sdk` list and
the `firebase-ios-sdk` dependency are the concrete results of working
through Part A.4 below for this module's specific SDK set — if a future
version bump introduces a *new* name collision, the error message will name
it and you add it here.

```sh
#!/usr/bin/env bash
# Regenerates Package.swift from whatever *.xcframework bundles are present in
# Frameworks/ (a symlink to ../FlutterSDK/Frameworks, populated by
# `flutter build ios-framework`) and VendoredThirdParty/ (real dynamic
# xcframeworks for SDKs that aren't part of Flutter's own build — see
# README.md for how each one was produced). Re-run after every framework
# rebuild — the plugin set changes release to release and hand-maintaining
# the binaryTarget list isn't worth it.
set -euo pipefail

cd "$(dirname "$0")"

OUTPUT="Package.swift"
DIRS=("Frameworks" "VendoredThirdParty")

# firebase-ios-sdk's transitive dependencies (grpc-binary, promises,
# abseil-cpp-binary, leveldb, GoogleDataTransport, etc.) declare targets with
# these exact names regardless of which of firebase-ios-sdk's *products* we
# actually depend on — Xcode's full project-level resolution checks
# target-name uniqueness across every target declared by every package
# present in the graph, not just the active build plan. Since we vendor
# Flutter's own (or our own one-time forced-dynamic builds of) already-correct
# binaries for these under different SwiftPM-internal identifiers, rename
# just the target name SwiftPM sees — the underlying .xcframework file (and
# its baked-in module/install name) is untouched.
collides_with_firebase_sdk=(
  FirebaseAppCheckInterop FirebaseCoreExtension FirebaseCrashlytics
  FirebaseFirestore FirebaseFirestoreInternal FirebaseMessaging
  FirebaseRemoteConfigInterop FirebaseSessions FirebaseSharedSwift
  GoogleDataTransport Promises FBLPromises absl grpc grpcpp openssl_grpc leveldb
  nanopb FirebaseCore FirebaseCoreInternal FirebaseInstallations
)

target_name_for() {
  local xcframework_name="$1"
  for c in "${collides_with_firebase_sdk[@]}"; do
    if [ "$c" = "$xcframework_name" ]; then
      echo "Vendored_${xcframework_name}"
      return
    fi
  done
  echo "$xcframework_name"
}

names=()
for dir in "${DIRS[@]}"; do
  [ -d "$dir" ] || continue
  for path in "$dir"/*.xcframework; do
    [ -e "$path" ] || continue
    names+=("$(basename "$path" .xcframework):$dir")
  done
done

if [ "${#names[@]}" -eq 0 ]; then
  echo "error: no .xcframework bundles found in ${DIRS[*]} — build the framework first" >&2
  exit 1
fi

{
  echo "// swift-tools-version:5.9"
  echo "// GENERATED by generate_package_swift.sh — do not edit by hand, re-run the script instead."
  echo "import PackageDescription"
  echo
  echo "let package = Package("
  echo "    name: \"FlutterSDK\","
  echo "    platforms: [.iOS(.v15)],"
  echo "    products: ["
  echo "        .library(name: \"FlutterSDK\", targets: [\"FlutterSDK\"]),"
  echo "    ],"
  echo "    dependencies: ["
  echo "        .package(url: \"https://github.com/firebase/firebase-ios-sdk\", exact: \"12.13.0\"),"
  echo "    ],"
  echo "    targets: ["

  for entry in "${names[@]}"; do
    xcframework_name="${entry%%:*}"
    dir="${entry##*:}"
    target_name="$(target_name_for "$xcframework_name")"
    echo "        .binaryTarget(name: \"${target_name}\", path: \"${dir}/${xcframework_name}.xcframework\"),"
  done

  echo "        .target("
  echo "            name: \"FlutterSDK\","
  echo "            dependencies: ["
  for entry in "${names[@]}"; do
    xcframework_name="${entry%%:*}"
    target_name="$(target_name_for "$xcframework_name")"
    echo "                .target(name: \"${target_name}\"),"
  done
  echo "                .product(name: \"FirebaseAnalytics\", package: \"firebase-ios-sdk\"),"
  echo "            ]"
  echo "        ),"
  echo "    ]"
  echo ")"
} > "$OUTPUT"

echo "Wrote $OUTPUT with ${#names[@]} binary targets."
```

Run it: `./generate_package_swift.sh`. At this point `Package.swift` exists
but the package won't run correctly until `VendoredThirdParty/` is
populated — proceed to A.4.

## A.4 Handle SDKs that Flutter's build doesn't vendor correctly

For this module, ten SDKs needed one of three fixes below:
`FirebaseAnalytics`, `GoogleMaps`, `AdjustSdk`, `AdjustSigSdk`,
`FBLPromises`, `nanopb`, `GoogleUtilities`, `FirebaseCore`,
`FirebaseCoreInternal`, and `FirebaseInstallations`. These are already
vendored in `VendoredThirdParty/` — you only need to redo this section on a
version bump (new plugin, new Flutter/Firebase/Adjust/Google Maps version)
that changes what's missing. The method to (re)diagnose:

1. Open the package in Xcode (Add Local Package to a throwaway/real host
   app), build, run.
2. If it crashes at launch with `dyld: Library not loaded:
   @rpath/Foo.framework/Foo`, something Flutter vendored expects a
   dynamically-loadable `Foo` that doesn't exist anywhere in the package
   yet. Go to A.4.1.
3. Fix it, rebuild, run again. Repeat until it launches cleanly — dyld only
   reports the *first* broken dependency it finds, so fixing one commonly
   reveals the next.
4. Once it launches, run the **closure scan** (A.5) to confirm there's
   nothing else lurking, rather than continuing to wait for crashes one at
   a time.

The fix patterns below are written generically (with placeholder name
`Foo`) since the exact mechanics are reusable for whatever new SDK a future
version bump introduces, but everything actually vendored in this package
today is one of the ten names above.

### A.4.1 Diagnose what's actually missing

```sh
otool -L /path/to/Foo.xcframework/ios-arm64/Foo.framework/Foo
```

This shows you every dynamic library `Foo` depends on. Anything starting
`@rpath/` needs to either already exist in `Frameworks/`/`VendoredThirdParty/`,
or be added. Three possible fixes, in order of how much work they are:

### A.4.2 Fix pattern 1: it's an SPM package that already ships the right binary

Example: `FirebaseAnalytics`. Some SDKs ship as a precompiled `binaryTarget`
inside their own official SPM package — these embed correctly with zero
extra work. Just add the package dependency and product:

```swift
// In Package.swift's dependencies:
.package(url: "https://github.com/firebase/firebase-ios-sdk", exact: "12.13.0"),
// In FlutterSDK target's dependencies:
.product(name: "FirebaseAnalytics", package: "firebase-ios-sdk"),
```

**Then this product must also be added directly to the *consuming app's*
target** (Frameworks, Libraries, and Embedded Content → Embed & Sign) — a
product only depended on transitively through your package's umbrella
target doesn't get embedded by Xcode. Document this requirement for whoever
integrates the package (see Part B).

How to tell if an SDK qualifies for this pattern: add it as a dependency,
build, and check whether the product shows up as a real `.framework` in the
build output (`Build/Products/.../PackageFrameworks/Foo.framework`) with an
actual Mach-O dynamic library inside (`file` on the binary should say
"dynamically linked shared library", not "current ar archive").

### A.4.3 Fix pattern 2: it's a real binary, just packaged for CocoaPods

Example: `GoogleMaps`. Some SDKs ship a genuine precompiled xcframework, but
only through CocoaPods, with metadata SwiftPM's binaryTarget loader is
stricter about than Xcode's native CocoaPods integration.

1. Find it: after any local `pod install` that resolved this pod (even in
   an unrelated project on the same machine), it's cached at
   `~/Library/Caches/CocoaPods/Pods/Release/<PodName>/<version>-<hash>/`.
   Look for a `Frameworks/Foo.xcframework` inside.
2. Copy it into `VendoredThirdParty/Foo.xcframework`.
3. Fix common metadata issues:
   - `'HeadersPath' is not supported for a 'framework'` build error: the
     xcframework's `Info.plist` has a redundant `HeadersPath` key CocoaPods
     adds but SwiftPM rejects. Remove it:
     ```sh
     /usr/libexec/PlistBuddy -c "Delete :AvailableLibraries:0:HeadersPath" \
       -c "Delete :AvailableLibraries:1:HeadersPath" Foo.xcframework/Info.plist
     ```
     (add/remove indices to match however many `AvailableLibraries` entries
     exist).
   - `Framework ... did not contain an Info.plist` embed error: CocoaPods
     doesn't require the *inner* `<slice>/Foo.framework/Info.plist` for
     static-only embedding; Xcode's embed step does. Synthesize a minimal
     one for each slice (`ios-arm64`, `ios-arm64_x86_64-simulator`, etc.):
     ```
     <?xml version="1.0" encoding="UTF-8"?>
     <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
     <plist version="1.0"><dict>
       <key>CFBundleExecutable</key><string>Foo</string>
       <key>CFBundleIdentifier</key><string>com.example.Foo</string>
       <key>CFBundleInfoDictionaryVersion</key><string>6.0</string>
       <key>CFBundleName</key><string>Foo</string>
       <key>CFBundlePackageType</key><string>FMWK</string>
       <key>CFBundleShortVersionString</key><string>1.0.0</string>
       <key>CFBundleVersion</key><string>1.0.0</string>
       <key>MinimumOSVersion</key><string>15.0</string>
     </dict></plist>
     ```
4. Check with `otool -L`/`file` whether the binary inside is actually
   dynamic or static (`current ar archive` = static). If static, like
   `GoogleMaps` turned out to be, you're done — it just needs to be a valid,
   linkable bundle, no further action. If genuinely dynamic, confirm the
   install name matches what depends on it (`otool -D`).

### A.4.4 Fix pattern 3: no binary exists anywhere — compile it yourself

Example: `AdjustSdk`, `FBLPromises`, `nanopb`, `GoogleUtilities`,
`FirebaseCore`, `FirebaseCoreInternal`, `FirebaseInstallations`. Some SDKs
ship as plain source under both CocoaPods and SPM — every distribution
channel compiles fresh with SPM's ambiguous "automatic" linkage, which
resolves to a static archive with a single consumer, never satisfying a
hard-coded dynamic `@rpath` dependency. The fix: compile it yourself, once,
with `type: .dynamic` forced, using Xcode's native SPM build support
directly against the source — no CocoaPods integration step, no scratch
Xcode project, nothing for the integrating app to ever see.

1. **Get the source.** After any local `pod install` that resolved this
   pod, it's cached as plain source at
   `~/Library/Caches/CocoaPods/Pods/Release/<PodName>/<version>-<hash>/`.
   Copy the relevant `.h`/`.m`/`.swift` files into a scratch
   `Sources/Foo/` folder inside your package, preserving whatever nested
   folder structure the original imports expect (see the import-style note
   below — this matters).
2. **Add a throwaway target + product** to `Package.swift` forcing dynamic
   linkage:
   ```swift
   .library(name: "Foo", type: .dynamic, targets: ["Foo"]),
   // ...
   .target(name: "Foo", path: "Sources/Foo", /* cSettings etc. as needed */),
   ```
   If `Foo` collides with a target name some external dependency (like
   `firebase-ios-sdk`) declares transitively, use a different scratch name
   (e.g. `FooBuild`) and rename the *compiled framework* back to `Foo`
   afterward (step 4) — `swift package resolve` will tell you immediately
   if you have a collision, with a "multiple packages ... declare targets
   with a conflicting name" error. **Exception: if `Foo` has actual Swift
   source, do not use this rename trick — see the Swift-specific note
   below.**
3. **Build directly against the package** — no host app project needed,
   Xcode auto-generates a scheme per product:
   ```sh
   xcodebuild -scheme Foo -destination "generic/platform=iOS Simulator" -configuration Release -derivedDataPath /tmp/foo-build build
   xcodebuild -scheme Foo -destination "generic/platform=iOS" -configuration Release -derivedDataPath /tmp/foo-build build
   ```
   Fix whatever compile errors come up (see "Common compile errors" below —
   these are near-universal for CocoaPods-sourced ObjC/Swift code built via
   plain SPM).
4. **If you used a scratch name**, rename the built `.framework` bundle and
   its inner binary back to the real name, fix the install name, and
   re-sign:
   ```sh
   mv FooBuild.framework Foo.framework
   mv Foo.framework/FooBuild Foo.framework/Foo
   install_name_tool -id "@rpath/Foo.framework/Foo" Foo.framework/Foo
   /usr/libexec/PlistBuddy -c "Set :CFBundleExecutable Foo" Foo.framework/Info.plist
   codesign --force -s - Foo.framework
   ```
5. **Merge device + simulator into one xcframework:**
   ```sh
   xcodebuild -create-xcframework \
     -framework /tmp/foo-build/Build/Products/Release-iphoneos/PackageFrameworks/Foo.framework \
     -framework /tmp/foo-build/Build/Products/Release-iphonesimulator/PackageFrameworks/Foo.framework \
     -output VendoredThirdParty/Foo.xcframework
   ```
6. **Delete the throwaway source/manifest changes** — only the resulting
   `.xcframework` needs to live in `VendoredThirdParty/`; re-run
   `generate_package_swift.sh` to pick it up automatically as a plain
   `binaryTarget`. Add its name to `collides_with_external_deps` in the
   generator if it collided with something in step 2.

**Common compile errors and fixes**, roughly in the order you'll hit them:

- **Quoted imports like `#import "Foo/Bar/Public/Foo/Baz.h"`**: these
  expect the *original nested CocoaPods folder layout* preserved exactly
  under your target's `path:`, not flattened. Copy preserving the full
  subdirectory structure, and add `cSettings: [.headerSearchPath(".")]`
  (the target root) so quoted full-path imports resolve.
- **Angle-bracket imports like `#import <Foo/Bar.h>`** (often in *other*
  files of the *same* pod, alongside the quoted style above): these expect
  a flat `Foo/Bar.h` reachable via a header search path. Satisfy both
  styles at once by making your target's `publicHeadersPath` folder contain
  a subfolder named after the module (`include/Foo/*.h`) **as symlinks to
  the original nested headers, not copies** — copies of the same header
  reachable two ways in one build can trigger "duplicate interface
  definition" errors during module validation.
- **"public headers (include) directory path ... is invalid"**: a
  `type: .dynamic` library product requires SwiftPM to know which headers
  are public. Add an `include/` folder (real or symlinked, per above) and
  set `publicHeadersPath: "include"`.
- **Only symlink headers that live in the pod's own `Public/` folder** —
  pulling private/internal headers into `publicHeadersPath` makes the
  auto-generated Clang module fail to find *their* nested quoted imports
  (each public header gets validated in isolation).
- **A pod with multiple CocoaPods subspecs** (e.g. `GoogleUtilities` ships
  ~9: `AppDelegateSwizzler`, `Environment`, `Logger`, etc.) that CocoaPods
  combines into *one* framework when several are pulled in together:
  combine them into one SPM target too, matching the single
  `@rpath/Foo.framework/Foo` dependency that's actually needed — don't
  follow an upstream SPM manifest that splits them into per-subspec
  targets/products, if one exists; it won't match what's needed here.
- **`@import Foo;` / `internal import Foo` Clang-module-style imports**
  (as opposed to `#import`) referencing the *real* product name when you
  used a scratch name in step 2: patch these source files directly to use
  your scratch name instead (safe — these copies are scratch-only, deleted
  in step 6).
- **A version-specific compiler define** (e.g. Firebase's `FIRVersion.m`
  needs `Firebase_VERSION` defined): check the error message and add the
  needed `cSettings: [.define("X", to: "\"value\"")]`.
- **Cross-target header references** (e.g. one scratch target's quoted
  import reaches into a sibling scratch target's folder): add a relative
  `.headerSearchPath("../OtherTarget")`.
- **A binaryTarget dependency (e.g. an SDK you already fixed in a prior
  pass) needed via `#import`/`import`, not just linking**: binaryTargets
  built via this same recipe don't get a `Headers/` folder by default (not
  needed for *linking*, only for being imported by name at *compile time* by
  something else later) — add one manually:
  ```sh
  mkdir -p Foo.xcframework/<slice>/Foo.framework/Headers
  cp /path/to/original/pod/public/headers/*.h Foo.xcframework/<slice>/Foo.framework/Headers/
  ```
  If something needs to Swift-`import` it (not just `#import`), it also
  needs an actual Clang module, not just loose headers — add
  `Modules/module.modulemap`:
  ```
  framework module Foo {
    umbrella "Headers"
    export *
    module * { export * }
  }
  ```

### A.4.5 The Swift-specific gotcha: module names are baked into symbols

**If the target you're compiling has actual Swift source (not just
Objective-C), the "rename the scratch-built framework afterward" trick in
A.4.4 step 4 does not work — and the failure mode is much more confusing
than a build error.**

Objective-C symbols don't encode the module name, so renaming a compiled
`.framework` bundle after the fact is safe. Swift symbols do — the module
name is mangled directly into every symbol
(`_$s<length><ModuleName><rest>`). If you build a Swift target as scratch
name `FooBuild` and rename the *file* to `Foo.framework` afterward, the
*symbols inside* still say `FooBuild` — this builds and links without any
warning (Xcode doesn't check this), then fails only at **launch**, with:

```
dyld: Symbol not found: _$s20RealModuleName10SomeClassC...
Expected in: .../Foo.framework/Foo
```

The fix: build any Swift target in a **fully isolated** scratch package —
its own temp directory, its own minimal `Package.swift`, with **no**
dependency on the package that caused the collision in scope at all (e.g.
no `firebase-ios-sdk` dependency declared) — so there's nothing to collide
with and you can use the real target name from the start:

```swift
// swift-tools-version:5.9
import PackageDescription
let package = Package(
    name: "FooIsolated",
    platforms: [.iOS(.v15)],
    products: [.library(name: "Foo", type: .dynamic, targets: ["Foo"])],
    targets: [
        // Reference your own already-vendored dependencies as binaryTargets
        // by relative path, not by re-declaring the colliding package:
        .binaryTarget(name: "SomeDependency", path: "SomeDependency.xcframework"),
        .target(name: "Foo", dependencies: ["SomeDependency"], path: "Sources/Foo"),
    ]
)
```

Build it (`xcodebuild -scheme FooIsolated ...`, same as A.4.4 step 3) — the
scheme name here will be the *package* name since there's only one product.
No rename step is needed this time since you used the real name from the
start. Before merging into an xcframework, verify a known symbol carries
the right module name:

```sh
nm -gU Foo.framework/Foo | grep SomeKnownSymbol | xcrun swift-demangle
```

The output should show your real module name, not a scratch placeholder.

One more wrinkle: Objective-C targets that merely `@import`/`#import` your
renamed Swift framework (rather than containing Swift themselves) may turn
out fine *without* rebuilding — check with `otool -L Foo.framework/Foo |
grep TheDependency` first; if no `LC_LOAD_DYLIB` line shows up, that
consumer never had a hard runtime dependency on the wrong name and doesn't
need rebuilding.

## A.5 Verify completeness

Don't wait for the next `dyld` crash one at a time — scan every vendored
xcframework's binary for `@rpath` dependencies and diff against what's
actually vendored:

```sh
have=(); for d in Frameworks VendoredThirdParty; do for f in "$d"/*.xcframework; do
  have+=("$(basename "$f" .xcframework)"); done; done

needed=(); for d in Frameworks VendoredThirdParty; do for f in "$d"/*.xcframework; do
  name="$(basename "$f" .xcframework)"
  bin="$f/ios-arm64/${name}.framework/${name}"
  [ -f "$bin" ] || continue
  needed+=($(otool -L "$bin" | grep "@rpath" | awk '{print $1}' \
    | sed -E 's#@rpath/([^/]+)\.framework/.*#\1#' | grep -v '\.dylib$'))
done; done

comm -23 <(printf '%s\n' "${needed[@]}" | sort -u) <(printf '%s\n' "${have[@]}" | sort -u)
```

Empty output = closure complete. (`libswift*.dylib` entries are filtered out
— those are normal system Swift runtime libraries, not something you
vendor.) Then do a full build directly against the package to catch
anything the scan can't (target-name collisions, the Swift symbol-mangling
issue from A.4.5):

```sh
xcodebuild -scheme FlutterSDK -destination "generic/platform=iOS Simulator" \
  -derivedDataPath /tmp/full-build build
```

Run this scan again after every `flutter build ios-framework` refresh or
dependency version bump — a new Flutter plugin or transitive pod can
introduce a new gap the same way this project's cluster of 7 turned up.

## A.6 Reference: this project's final package layout

```
FlutterSDKPackage/
├── Package.swift                          (generated, ~65 binaryTargets)
├── generate_package_swift.sh
├── Frameworks/ → ../FlutterSDK/Frameworks  (symlink; Flutter's own build output)
└── VendoredThirdParty/
    ├── GoogleMaps.xcframework              (real binary, lifted from CocoaPods cache)
    ├── AdjustSdk.xcframework                (compiled from source, forced dynamic)
    ├── AdjustSigSdk.xcframework             (real binary, lifted from CocoaPods cache)
    ├── FBLPromises.xcframework              (compiled from source, forced dynamic)
    ├── nanopb.xcframework                   (compiled from source, forced dynamic)
    ├── GoogleUtilities.xcframework          (compiled from source, forced dynamic)
    ├── FirebaseCore.xcframework             (compiled from source, forced dynamic)
    ├── FirebaseCoreInternal.xcframework      (compiled from source, forced dynamic, Swift)
    └── FirebaseInstallations.xcframework    (compiled from source, forced dynamic)
```

Only `firebase-ios-sdk` (for the `FirebaseAnalytics` product) remains as an
external SPM package dependency — everything else is self-contained.

---

# Part B — Integrating into a host iOS app

## Prerequisites

- iOS 15.0+, Swift 5.9+, Xcode 15+

## Step 1: Add the package

1. Xcode → **File → Add Package Dependencies… → Add Local…** → select the
   `FlutterSDKPackage` folder (or point at a remote git URL/tag once
   published to its own repo).
2. Add the `FlutterSDK` library product to your app target.

## Step 2: Add Firebase directly to your app target

**Required, not optional.** Project → your app target → **Frameworks,
Libraries, and Embedded Content** → **+** → pick the `Firebase` package
(already pulled in transitively by `FlutterSDKPackage`, so it'll appear in
the picker) → product `FirebaseAnalytics` → **Embed & Sign**.

Why: a package product depended on only *transitively* (through
`FlutterSDKPackage`'s own umbrella target) isn't embedded by Xcode — it
needs to be a *direct* dependency of the final app target. `GoogleMaps`,
`Adjust`, and the rest of the Firebase ecosystem don't need this because
they're vendored as real binaries inside the package, not resolved via an
SPM source package.

## Step 3: Add the `-ObjC` linker flag

App target → **Build Settings** → **Other Linker Flags** → add `-ObjC`.

SPM has no equivalent of CocoaPods' automatic linker-flag injection for a
package's consumer, so this has to be set manually. Firebase's own SPM docs
call out the same requirement.

## Step 4: Google Maps API key

If your integration uses flows that render maps, provide your API key
before anything else touches the SDK — typically in
`AppDelegate.application(_:didFinishLaunchingWithOptions:)`:

```swift
import GoogleMaps
GMSServices.provideAPIKey("YOUR_API_KEY_HERE")
```

## Step 5: Build the native host bridge

The bridge class, method channel contract
(`initialize`/`openBookingFlow`/`openOrderFlow`/`hostReady`/`setUserData`/
`closeSDK`), data model, payment integration, and lifecycle handling are
identical to a CocoaPods integration and don't depend on anything in this
guide. See "Step 2: Create Native Host Bridge" onward in
`NATIVE_INTEGRATION_GUIDE.md` (in the SDK repo), or copy
`SgTravelNative.swift` from
`native_example/TravelSDKiOSDemo/TravelSDKiOSDemo/` in the demo app as a
working starting point.

---

# Part C — Troubleshooting

**Problem**: `dyld: Library not loaded: @rpath/Foo.framework/Foo` at
launch, for any framework name, even though the build succeeded
- **Cause**: `Foo` is missing entirely, or was built as a static archive
  when something expects it dynamically.
- **Solution**: Part A.4 of this guide — diagnose with `otool -L`, fix with
  whichever of the three patterns applies, then re-run the closure scan
  (A.5).

**Problem**: `dyld: Symbol not found` at launch, mentioning a Swift mangled
symbol (starts with `_$s`), even though `Foo.framework` itself loads fine
- **Cause**: A Swift framework was rebuilt under a different internal
  module name than what consumes it expects (Swift bakes the module name
  into every symbol, unlike Objective-C).
- **Solution**: Part A.4.5 — rebuild the Swift target in a fully isolated
  package using its real name from the start.

**Problem**: "Missing package product" errors for everything, even products
that were previously resolving fine
- **Cause**: Usually a stale `Package.resolved` (in
  `YourApp.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/`) referencing
  a dependency that's no longer declared, or a genuine resolution failure
  upstream that cascades into every product showing as unresolved.
- **Solution**: Xcode → **File → Packages → Reset Package Caches**, then
  **Resolve Package Versions**. Check the Report Navigator's full
  resolution log (not just the Issue Navigator summary) for the real
  underlying error.

**Problem**: `multiple packages declare targets with a conflicting name: 'Foo'`
- **Cause**: An external dependency (or one of its own transitive
  dependencies) declares a target with the same name as one of your
  vendored binaryTargets. Xcode's full project-level resolution checks
  target-name uniqueness across every target declared by every package
  present in the graph — not just the ones in the active build plan, so a
  plain `swift package resolve` on the package alone won't catch this ahead
  of time.
- **Solution**: Add the colliding name to `collides_with_external_deps` in
  `generate_package_swift.sh`, regenerate.

**Problem**: "Engine destruction crashes app"
- **Solution**: Use weak references (`[weak self]`) in closures to prevent
  retain cycles (applies regardless of CocoaPods vs SPM).
