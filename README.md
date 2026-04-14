# bareBonesBlue-os &nbsp; [![bluebuild build badge](https://github.com/hovarak/barebonesblue-os/actions/workflows/build.yml/badge.svg)](https://github.com/hovarak/barebonesblue-os/actions/workflows/build.yml)

`bareBonesBlue-os` is a custom Fedora Atomic desktop image built with [BlueBuild](https://blue-build.org/). The goal is to define the operating system declaratively, keep it in git, and rebuild it consistently instead of manually re-configuring machines after every install.

This repository treats the desktop OS like a versioned artifact:

- the base image is inherited from a BlueBuild-compatible atomic desktop image
- the final system is assembled from a small YAML recipe
- package choices, default services, and Flatpaks are reviewed as code
- builds are published automatically through GitHub Actions

The current image uses `ghcr.io/wayblueorg/hyprland-nvidia-open` as its base and layers personal packages, enabled services, and desktop applications on top.

## Project structure

This repository is intentionally kept small. The main files are:

```text
.
├── .github/workflows/build.yml
├── cosign.pub
├── files/
│   └── README.md
├── recipe.yml
└── README.md
```

What each part does:

- `recipe.yml`: the single source of truth for the image definition. This is where the base image, RPM packages, enabled services, Flatpaks, and signing module are declared.
- `.github/workflows/build.yml`: builds and publishes the image in GitHub Actions. It points directly at `recipe.yml`.
- `cosign.pub`: the public key used to verify signed images.
- `files/`: optional overlay content. It is not active right now. Add a `files` module back to `recipe.yml` when you want to copy files like `/etc` configuration, branding assets, or custom scripts into the image.

## How the image model works

BlueBuild uses a recipe-first workflow:

1. `recipe.yml` declares the image you want.
2. BlueBuild generates a temporary `Containerfile` from that recipe.
3. The container build produces an OCI image that Fedora Atomic systems can boot.
4. GitHub Actions can publish that image to a registry such as GHCR.
5. Fedora Silverblue and other atomic Fedora variants can rebase onto the published image.

In this repository, `recipe.yml` is organized into a small set of modules:

- `dnf`: installs RPM packages like `neovim`, `zsh`, `tailscale`, Hyprland utilities, and fonts; it also removes inherited packages that are not wanted in the final image.
- `systemd`: enables services that should be on by default, currently Bluetooth and Tailscale.
- `default-flatpaks@v2`: installs the desktop application set system-wide.
- `signing`: signs the output image so rebases can be verified.

That means the repo does not directly mutate a live system. Instead, it defines what the next immutable image should contain.

## Running locally on Fedora Silverblue

These steps are best done on a Fedora Silverblue machine or a disposable Fedora Silverblue VM. Testing in a VM first is the safer workflow.

### 1. Install or access BlueBuild CLI

BlueBuild documents local builds in its official guide:

- Local builds and testing: <https://blue-build.org/how-to/local/>

If you are already running a BlueBuild-based image, the `bluebuild` CLI is installed by default. Otherwise, install it using the method documented in the BlueBuild CLI instructions.

### 2. Clone this repository

```bash
git clone https://github.com/hovarak/barebonesblue-os.git
cd barebonesblue-os
```

### 3. Preview the generated Containerfile

This is useful when you want to inspect exactly what BlueBuild will build.

```bash
bluebuild generate ./recipe.yml -o Containerfile
```

The generated `Containerfile` is ignored by git in this repository because it is build output, not source.

### 4. Build the image locally

```bash
bluebuild build ./recipe.yml
```

This validates that the recipe builds locally before you push changes to GitHub.

### 5. Rebase a Fedora Silverblue test machine onto the locally built image

BlueBuild provides a workflow command for this:

```bash
sudo bluebuild switch ./recipe.yml
```

That command builds the image, exports it, and performs the `rpm-ostree` rebase locally. On a real workstation, use a test deployment or a VM first.

## Running virtually in a VM

The simplest virtual test workflow is to generate an ISO from this repository and boot it in a VM manager such as GNOME Boxes or `virt-manager`.

Official BlueBuild ISO docs:

- ISO generation: <https://blue-build.org/how-to/generate-iso/>

To build an ISO directly from this local recipe:

```bash
sudo bluebuild generate-iso --iso-name bareBonesBlue-os.iso recipe ./recipe.yml
```

This produces `bareBonesBlue-os.iso` in the current working directory. Create a new VM, attach that ISO, and install it the same way you would install any Fedora Atomic desktop.

If you prefer to test the already published image instead of building from source locally, BlueBuild also supports generating an ISO from a remote image.

## Rebasing an existing Fedora Silverblue install to the published image

> [!WARNING]
> Fedora native container rebasing is still considered experimental upstream. Test carefully and keep a rollback path.

If you want to switch an existing atomic Fedora installation to the published image from GHCR:

```bash
rpm-ostree rebase ostree-unverified-registry:ghcr.io/hovarak/barebonesblue-os:latest
systemctl reboot
rpm-ostree rebase ostree-image-signed:docker://ghcr.io/hovarak/barebonesblue-os:latest
systemctl reboot
```

The `latest` tag follows the latest successful build for this repository.

## Verification

These images are signed with [cosign](https://github.com/sigstore/cosign). Verify the published image with:

```bash
cosign verify --key cosign.pub ghcr.io/hovarak/barebonesblue-os
```
