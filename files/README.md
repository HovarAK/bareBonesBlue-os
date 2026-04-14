# `files/`

This directory is optional.

Add it back to `recipe.yml` with a `files` module when you want to copy custom files into the image, for example:

- `/etc` configuration
- desktop branding assets
- system scripts or wrappers

Suggested layout:

```text
files/
  system/
    etc/
    usr/
```

BlueBuild copies the contents of `files/system/` into `/` inside the built image.
