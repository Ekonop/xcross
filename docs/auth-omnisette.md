# GrandSlam -80046 authentication fix (Omnisette)

This document describes the fork/branch that fixes Apple GrandSlam
authentication error `-80046` by switching from the legacy Android ADI
anisette implementation to a remote macOS Omnisette provider.

## Why this change exists

Since October 2026 Apple's GrandSlam rejects Android-identifying clients
with error `-80046`. The previous implementation used Android ADI
libraries (`libCoreADI.so`, `libstoreservicescore.so`) and could no longer
sign in. The fix ports the macOS Omnisette approach from xtool PR #380 and
xcross PR #139.

## Implementation

- `OmnisetteAnisetteProvider` implements the v3 Omnisette protocol:
  - `GET /v3/client_info`
  - `WS /v3/provisioning_session`
  - `POST /v3/get_headers`
- Apple's `midStartProvisioning` / `midFinishProvisioning` plist dance is
  still driven by xcross, while anisette material is synthesized by the
  remote provider.
- `AnisetteState` stores `adiPb` and `providerId`. State provisioned by
  the old Android backend is reset and reprovisioned automatically.
- GrandSlam two-factor requests use the `akd` User-Agent.
- Legacy saved Apple ID sessions remain readable, but no longer write
  `adiLibraryDirectory`.

## Provider configuration and privacy

Default provider:

```
https://mac-ani.jkcoxson.com
```

This is a third-party service, not an Apple-operated endpoint. Your Apple
ID password and 2FA code are sent only to Apple. The provider can observe
device identifiers, provisioning material, request metadata, and anisette
headers, but not your credentials.

Override the provider:

```
XCROSS_ANISETTE_URL=https://your-trusted-provider.example xcross auth --apple-id you@example.com
```

The selected provider and endpoint are printed by `xcross auth`. TLS
verification is never disabled. No credentials are embedded in source code.

## Building the development version

```
cd /home/vansh/xcross
dart pub get
cd packages/xcross
dart run -DXCROSS_VERSION=unreleased -DXCROSS_RELEASED=false tool/build_xcross.dart
```

Install the resulting bundle separately, for example:

```
mkdir -p ~/.local/xcross-dev/bin ~/.local/xcross-dev/lib
cp -a packages/xcross/build/cli/linux_x64/bundle/bin/. ~/.local/xcross-dev/bin/
cp -a packages/xcross/build/cli/linux_x64/bundle/lib/. ~/.local/xcross-dev/lib/
mv ~/.local/xcross-dev/bin/xcross ~/.local/xcross-dev/bin/xcross-dev
ln -sf ~/.local/xcross-dev/bin/xcross-dev ~/.local/bin/xcross-dev
```

## Release builds

Release/profile builds require an iOS `gen_snapshot` AOT compiler. The
`xcross_gen_snapshot` project currently publishes one for Flutter 3.47.6
(engine `692136cb`), but not for Flutter 3.47.7 (engine `deb28748`).

To make release builds work on this machine, use the separately cloned
Flutter 3.47.6 SDK without changing the main Flutter 3.47.7:

```
export PATH="/home/vansh/flutter-3.47.6/bin:/home/vansh/.local/bin:/home/vansh/.local/share/swiftly/bin:$PATH"
export LD_LIBRARY_PATH="/home/vansh/.local/lib:${LD_LIBRARY_PATH:-}"
source /home/vansh/.local/share/swiftly/env.sh
cd /home/vansh/xcross_smoke_test
xcross-dev flutter build --release
```

The resulting app is written under `build/xcross-ios/`. Add `--ipa` to
produce an `.ipa`. Switch back to Flutter 3.47.7 for debug work by
restoring `/opt/flutter` (or the default Flutter SDK) to `PATH`.


- `dart analyze` for `apple_developer_kit` and `xcross`: pass
- `dart test` for `apple_developer_kit`: pass
- `dart test` for `xcross`: pass after initializing submodules
- `omnisette_provider_test.dart`: pass
- Dev build `xcross-dev --version`: `xcross unreleased (unreleased build)`
- Apple authentication with the dev build: success
- Debug iOS build/sign/install: success
- Automatic device launch: failed via pymobiledevice3 process control
- Manual app open: works and shows the expected Flutter debug-mode message

## Known limitations

- Automatic launch on the tested iPhone (iOS 27.0.1) fails through
  pymobiledevice3; installation and signing succeed.
- Release/profile builds need the AOT compiler, which is not published for
  Flutter 3.47.7 on linux-x64.
- The default Omnisette provider is third-party; use `XCROSS_ANISETTE_URL`
  for a self-hosted or otherwise trusted compatible provider.
