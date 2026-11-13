# PhotoCraft for macOS 10.15 Catalina (unofficial)

Unofficial community project that makes [PhotoCraft](https://github.com/storytold/photocraft)
run on **macOS 10.15 Catalina on Intel Macs**. It is not affiliated with or endorsed by the
original authors. PhotoCraft is made by the ArtCraft team and its contributors and is licensed
under MIT OR Apache-2.0.

This is a GitHub fork of PhotoCraft. Its source code is **not modified**: the changes below are
applied at build time. What this fork adds:

- `screen-capture-catalina.patch`, a small source patch (see below),
- `.github/workflows/catalina.yml`, a GitHub Actions workflow that builds upstream PhotoCraft with
  the changes below,
- this README.

Builds are **Intel (x86_64) only**, **ad-hoc signed and not notarized**. Apple silicon Macs do not
need this project: the original app runs there on macOS 11 and newer.

## What is changed

Compared with the upstream release, only two things differ:

1. **Minimum macOS version lowered from 11.0 to 10.15.**
   `MACOSX_DEPLOYMENT_TARGET` in `packaging/macos/package.sh` and `LSMinimumSystemVersion` in
   `packaging/macos/Info.plist.in` are set to `10.15`.
2. **Screen color picker no longer crashes on 10.15.**
   "Select screen color" in the color dialog calls `CGPreflightScreenCaptureAccess` and
   `CGRequestScreenCaptureAccess`. Both were added in macOS 11, so on Catalina the app aborted
   with `dyld: Symbol not found`. The patch (`apps/photocraft/src/screen_color.rs`) skips these two
   calls on macOS 10.15. macOS asks for the Screen Recording permission itself the first time the
   screen is read. Nothing changes on macOS 11 and newer.

Nothing else is modified. Features that need newer system APIs than Catalina has may still fail
at runtime. Tested on macOS 10.15.x with PhotoCraft 0.23.1: start, opening and saving files,
screen color picker. Please report problems here, not to the original authors, since this build
is modified.

## Build it yourself (step by step)

Everything below is typed into the **Terminal** app (Applications > Utilities > Terminal).
Lines starting with `#` are comments and can be left out. The first build takes a long time
(30 minutes or more on an older Mac) and needs several GB of free disk space.

### 1. Apple developer tools

```
xcode-select --install
```

A window opens; click Install and wait. If it says the software is not available for Catalina,
download **Command Line Tools for Xcode 12.4** from
https://developer.apple.com/download/all/ (a free Apple ID is enough), install the `.dmg`, and
check with `xcrun --show-sdk-version`, which should print `10.15` or higher.

### 2. Rust

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Press Enter for the default installation, then **close the Terminal window and open a new one**.
Check with:

```
cargo --version
```

### 3. Get the code

```
mkdir -p ~/craft-build
cd ~/craft-build
git clone https://github.com/storytold/photocraft.git
git clone https://github.com/storytold/craft-i18n.git
git clone https://github.com/sudo-mk-sandwich/photocraft-catalina.git
cd photocraft
git checkout v0.23.1
```

`craft-i18n` holds the translations; without it the app is English only. `v0.23.1` is the version
this patch was tested with. Newer versions may work too, but the patch may no longer apply.

### 4. Apply the changes

```
git apply ../photocraft-catalina/screen-capture-catalina.patch
sed -i '' 's/MACOSX_DEPLOYMENT_TARGET=11\.0/MACOSX_DEPLOYMENT_TARGET=10.15/' packaging/macos/package.sh
perl -0pi -e 's|(LSMinimumSystemVersion</key>\s*<string>)11\.0|${1}10.15|' packaging/macos/Info.plist.in
```

Check that both lines changed:

```
git diff --stat
```

It should list `package.sh`, `Info.plist.in` and `screen_color.rs`.

### 5. Linker setting for older SDKs

If your developer tools contain the macOS 10.15 SDK, the linker does not know the two functions
mentioned above and the build stops with `Undefined symbols ... _CGPreflightScreenCaptureAccess`.
This setting tells it to resolve them at runtime (the patch makes sure they are only called where
they exist). Run the two lines one after the other:

```
mkdir -p .cargo
```

```
printf '[target.x86_64-apple-darwin]\nrustflags = [\n  "-C", "link-arg=-Wl,-U,_CGPreflightScreenCaptureAccess",\n  "-C", "link-arg=-Wl,-U,_CGRequestScreenCaptureAccess",\n]\n' > .cargo/config.toml
```

### 6. Build

```
CRAFT_I18N_DIR="$PWD/../craft-i18n" ./packaging/macos/package.sh --arch x86_64
```

Warnings are normal. Stop only at lines that start with `error`.

### 7. Where is the app?

- The app itself: `target/macos-package/PhotoCraft.app` inside the `photocraft` folder
  (`~/craft-build/photocraft/target/macos-package/PhotoCraft.app`). Open the folder with:

  ```
  open target/macos-package
  ```

- A disk image (`.dmg`) is also created. The last lines of the build output print its path.

Drag `PhotoCraft.app` into your Applications folder. Because the app is not signed by Apple,
open it the first time with **right-click > Open**, then confirm.

### Updating later

```
cd ~/craft-build/photocraft
git checkout -- .
git fetch --tags
git checkout <new version, e.g. v0.23.2>
git -C ../craft-i18n pull
git -C ../photocraft-catalina pull
```

Then repeat steps 4 to 6 (the `.cargo` folder from step 5 is kept, but `git checkout -- .` does not
remove it). If `git apply` reports an error, the new version changed the file the patch touches;
please open an issue here.

## License

The code that gets built is PhotoCraft under MIT OR Apache-2.0, see the upstream repository
for `LICENSE-MIT`, `LICENSE-APACHE` and `NOTICE`; those files are also bundled in the app. This
repository's own files (patch, workflow, README) are provided as is, without warranty.
