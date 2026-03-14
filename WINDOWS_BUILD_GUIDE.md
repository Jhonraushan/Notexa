# AppFlowy Windows Build Guide

> Tested on: Windows 11 x86_64
> Date: 2026-03-14
> Branch: `main`

---

## Prerequisites Checklist

| Tool | Required Version | Install Source |
|------|-----------------|----------------|
| Flutter | 3.27.4 | https://docs.flutter.dev/get-started/install/windows |
| Rust (via rustup) | 1.85 (toolchain auto-selected by `rust-toolchain.toml`) | https://win.rustup.rs/x86_64 |
| cargo-make | latest | `cargo install --force cargo-make` |
| duckscript_cli | latest | `cargo install --force duckscript_cli` |
| LLVM | 16.0.0 | https://github.com/llvm/llvm-project/releases/tag/llvmorg-16.0.0 |
| Visual Studio 2022 | Build Tools + "Desktop Development with C++" | https://visualstudio.microsoft.com/downloads/ |
| OpenSSL | 1.1.1 | https://slproweb.com/products/Win32OpenSSL.html |
| Perl (Strawberry) | x64 | https://strawberryperl.com/ |
| protoc_plugin | 21.1.2 | `dart pub global activate protoc_plugin 21.1.2` |
| vcpkg | latest | https://github.com/microsoft/vcpkg |

### Environment Variables Required

```
OPENSSL_DIR = C:\<your-openssl-install>\bin
PATH includes: C:\Windows\System32 (for PowerShell)
PATH includes: LLVM bin folder
PATH includes: vcpkg folder
```

### Flutter Configuration

```powershell
flutter config --enable-windows-desktop
flutter doctor   # fix any issues before proceeding
```

---

## Build Steps

### Step 1 — Clone the repository

```powershell
git clone https://github.com/AppFlowy-IO/appflowy.git
cd appflowy
```

### Step 2 — Full build (Rust + code generation + Flutter)

Run from **PowerShell or Windows cmd** (NOT Git Bash):

```powershell
cd frontend
cargo make --profile production-windows-x86 appflowy
```

This single command runs the following pipeline internally:

| Stage | What it does |
|-------|-------------|
| `appflowy-core-release` | Compiles the Rust backend → `dart_ffi.dll` |
| `code_generation` | Copies translations, generates protobuf `.pb.dart`, `locale_keys.g.dart`, freezed files, SVG classes |
| `set-app-version` | Stamps version info |
| `flutter-build` | Runs `flutter build windows --release` |
| `copy-to-product` | Copies output to `product/` folder |

**Expected build time:** 30–60 minutes (first run; Rust compilation is the bottleneck).

### Output Location

```
frontend/appflowy_flutter/build/windows/x64/runner/Release/AppFlowy.exe
```

---

## Issues Encountered & Fixes

### Issue 1 — `flutter build windows --release` alone fails

**Symptom:**
Running `flutter build windows --release` directly from `appflowy_flutter/` produces hundreds of errors:
```
Error when reading 'packages/appflowy_backend/lib/protobuf/flowy-error/errors.pb.dart':
  The system cannot find the path specified.
```

**Cause:**
The Flutter build depends on generated files that only exist after the Rust backend is compiled and code generation has run. `flutter build` alone skips both steps.

**Fix:**
Always use `cargo make --profile production-windows-x86 appflowy` from `frontend/`, never run `flutter build` in isolation.

---

### Issue 2 — `generate.cmd` runs silently and code generation is skipped

**Symptom:**
After `cargo make` finishes, `lib/generated/` directory does not exist. The `code_generation` task in the Makefile runs `scripts/code_generation/generate.cmd` via duckscript `exec`, but the command completes instantly with no output and no files generated.

**Cause:**
The `generate.cmd` script uses `xcopy` to copy `resources/translations/` to `assets/translations/`. When invoked from within Git Bash or duckscript, the `xcopy` call fails silently, which causes `easy_localization` to abort because `en-US.json` is not found.

**Fix (manual workaround):**
Run the code generation steps manually from inside `appflowy_flutter/`:

```bash
# 1. Copy resources
cp -r frontend/resources/translations/. frontend/appflowy_flutter/assets/translations/
cp -r frontend/resources/flowy_icons/.   frontend/appflowy_flutter/assets/flowy_icons/

# 2. Generate locale keys
cd frontend/appflowy_flutter
dart run easy_localization:generate -S assets/translations/
dart run easy_localization:generate -f keys -o locale_keys.g.dart -S assets/translations/ -s en-US.json

# 3. Generate freezed / json_serializable / envied files
dart run build_runner build -d

# 4. Generate FlowySvg icon classes
dart run flowy_svg
```

---

### Issue 3 — Sub-package freezed files not generated

**Symptom:**
After running `build_runner` for the main `appflowy_flutter` package, the build still fails with:
```
Error when reading 'packages/flowy_infra/lib/colorscheme/colorscheme.g.dart':
  The system cannot find the file specified.
Error when reading 'packages/flowy_infra/lib/plugins/bloc/dynamic_plugin_event.freezed.dart':
  The system cannot find the file specified.
```

**Cause:**
The `generate_freezed.cmd` script is supposed to run `build_runner` inside each sub-package under `appflowy_flutter/packages/`. This script was also skipped due to Issue 2. The `flowy_infra` package contains freezed and json_serializable annotated classes that require their own code generation pass.

**Fix:**
Run `build_runner` for each sub-package:

```bash
for pkg in appflowy_backend appflowy_popover appflowy_result appflowy_ui flowy_infra flowy_infra_ui flowy_svg; do
  dir="frontend/appflowy_flutter/packages/$pkg"
  if [ -f "$dir/pubspec.yaml" ]; then
    cd "$dir" && dart run build_runner build -d
  fi
done
```

---

### Issue 4 — `AFTextField` missing `autofillHints` parameter

**Symptom:**
Build fails with:
```
lib/workspace/presentation/settings/pages/account/password/change_password.dart(144,9):
  error GC6690633: No named parameter with the name 'autofillHints'.
```
Affects 8 call sites across 4 files:
- `change_password.dart`
- `setup_password.dart`
- `continue_with_password_page.dart`
- `set_new_password.dart`

**Cause:**
Commit [#8295](https://github.com/AppFlowy-IO/appflowy/pull/8295) (`fix: enable password manager autofill for all password fields`) added `autofillHints: const [AutofillHints.password]` to all password field call sites, but forgot to add the `autofillHints` parameter to the `AFTextField` widget itself in `packages/appflowy_ui/lib/src/component/textfield/textfield.dart`.

**Fix:**
Added `autofillHints` to `AFTextField` in [textfield.dart](frontend/appflowy_flutter/packages/appflowy_ui/lib/src/component/textfield/textfield.dart):

```dart
// In AFTextField constructor:
this.autofillHints,

// As a field:
final Iterable<String>? autofillHints;

// Passed to the inner TextField:
autofillHints: widget.autofillHints,
```

> **Note:** This fix should be submitted as a PR to the AppFlowy repository.

---

## Quick Reference — Manual Build Sequence

If `cargo make` fails partway through and you need to resume from the Flutter step:

```powershell
# Run in PowerShell from frontend/appflowy_flutter/

# Step A: Copy assets (if not done)
xcopy /E /Y /I ..\resources\translations assets\translations
xcopy /E /Y /I ..\resources\flowy_icons assets\flowy_icons

# Step B: Generate locale keys
dart run easy_localization:generate -S assets/translations/
dart run easy_localization:generate -f keys -o locale_keys.g.dart -S assets/translations/ -s en-US.json

# Step C: Generate freezed / json / envied
dart run build_runner build -d

# Step D: Generate SVG classes
dart run flowy_svg

# Step E: Generate sub-package code
cd packages\flowy_infra && dart run build_runner build -d && cd ..\..

# Step F: Build Flutter app
flutter build windows --release
```

---

## Notes

- **Run in PowerShell/cmd, not Git Bash** — the `OPENSSL_DIR` environment variable is only available to Windows-native processes. Git Bash does not inherit it from the Windows environment.
- **Rust toolchain** — the repo's `rust-toolchain.toml` pins to `1.85`. rustup will automatically download and use this version when building from the `frontend/` directory, even if your default toolchain is different.
- **First build is slow** — Rust compilation takes 30–45 min on first run. Subsequent builds are much faster due to incremental compilation.
