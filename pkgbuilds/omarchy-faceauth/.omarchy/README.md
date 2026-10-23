# omarchy-faceauth - face authentication on the IR camera

## Overview

`omarchy-faceauth` is the daemon that owns the IR camera and the enrolled face data, the `pam_faceauth.so` module and the `faceauth` command, built with `cargo build --frozen --release` from the tagged tarball of https://github.com/mellismas/omarchy-faceauth (Rust, MIT). The models it runs are the separate `omarchy-faceauth-models` package. Installing this package wires no PAM service and enables no unit: `omarchy setup security face` does that after a camera check, the models install, an enrolment and a verification, and `omarchy remove security face` takes it out again before the package goes.

## Hardware requirement, not a dependency

The daemon drives an ov7251 IR sensor on Intel's IPU3 pipeline and needs the kernel to expose the IR illuminator as a V4L2 strobe control, which it flashes in a random pattern to tell a face from a photo. The stock ov7251 driver lacks the control, so no shipped `linux-omarchy` or `linux-omarchy-bore` has it yet. The control is the last patch of a ten-patch linux-media series that has not been posted; the nine before it fix the IPU3 CIO2 driver and let the RGB camera stream alongside the IR one. On a kernel without the control the daemon refuses every attempt, the password is used, and setup stops before it downloads the models and says why. Pacman cannot express any of this, so it lives in the PKGBUILD comment and here. USB (UVC) IR modules, the kind most Windows Hello laptops use, are not supported. The reference machine, and the only one tested, is a Surface Book 2 (IPU3, ov7251).

## Why the models are their own package

The three ONNX files (YuNet detector, MIT; glintr100 recogniser, Apache-2.0; MediaPipe face mesh, Apache-2.0, converted from Google's TFLite file with a NOTICE saying so) are about 260 MB, so `omarchy-faceauth-models` is `arch=any`, built from the project's `models-N` GitHub release and watched by that tag. pacman owns, verifies and removes them, and `sha256sums` pins each file. The daemon's manifest, `/usr/share/faceauth/models.toml` in this package, carries the same checksums, and `faceauth doctor` verifies the installed files against it. A new release is a new number with new files, never a changed file under the old number.

The models are an `optdepends`, not a `depends`: setup installs this package first, runs `faceauth doctor` on the camera and the kernel, and installs the models only once that check passes, so a laptop that cannot use face authentication never downloads them. The daemon refuses to start without them.

## What install changes on its own

One thing: the udev rule `72-faceauth-ir.rules` makes the IR sensor's control node (exposure, gain, strobe) root-only. The RGB webcam is untouched. `post_install` says so, and `pre_remove` warns when a file under `/etc/pam.d` still names the module, since those files belong to other packages and only the Omarchy removal edits them.

## The lock helper's path

The daemon runs the walk-away lock helper, `faceauth-lock-session`, by a fixed path (`LOCK_HELPER` in the daemon source), so a recipe that installs it anywhere else leaves the daemon unable to lock. At v0.1.2 that path is `/usr/lib/faceauth/faceauth-lock-session`, which `package()` matches. The project's `packaging/release-check.sh` reads the constant from the tag and checks the recipe's install line against it, so a release that moves the helper fails the check until the recipe follows.

## The restart hook

`90-omarchy-faceauth-restart.hook` runs `systemd-hook restart faceauth.service` after an upgrade. It is numbered 90 so it runs after `30-systemd-daemon-reload-system.hook` has reloaded the unit files, and there is no `post_upgrade` because a restart from there would start the old unit. Arch's `systemd-hook` keeps a `restart` operation for other packages: it runs `try-restart`, which leaves a stopped or disabled service alone, and skips an upgrade inside a chroot or an unbooted root.

## What check() runs

`cargo test --frozen --release --workspace`, `bash tests/lock-session-test.sh`, and `bash tests/pam-module-test.sh target/release/libpam_faceauth.so` (the helper's test stubs `systemd-run` and needs no session). `makepkg` never runs as root and the sealing test needs a TPM and root, so the store test runs in its plaintext branch and a passing `check()` says nothing about templates sealing to the TPM. That is left to a supervised end-to-end test as root.

## Testing

`bash -n` and `makepkg --printsrcinfo` on both recipes; `updpkgsums` at the tag and `makepkg --verifysource` on the pinned sums; `packaging/release-check.sh` in the project tree, which rebuilds GitHub's tarball from the local tag and checks the pin, every file `package()` installs and `check()` runs, the helper path, the model checksums against the models recipe, and both watch patterns.

## Security-key and fingerprint helpers

The package installs the sandboxed key, fingerprint, and lock-state helpers under `/usr/lib/faceauth`, with their private PAM stacks, fingerprint D-Bus policy, and key-lease tmpfiles rule. `pam-u2f` supplies security-key authentication. Fingerprint authentication additionally requires the optional `fprintd` package. Installing these files does not enroll credentials or enable authentication in a system PAM stack.

## Release candidate

This recipe is prepared for v0.1.2. The tag and GitHub Release must be published, and the online release checks must pass, before this package can be built from the public download or submitted upstream.
