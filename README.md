# Heroku buildpack for ImageMagick 7.1, WebP, and HEIF

This is a [Heroku buildpack](http://devcenter.heroku.com/articles/buildpacks) for vendoring ImageMagick (with WebP and HEIF support) binaries into your project.

It supports the [Heroku stacks](https://devcenter.heroku.com/articles/stack) `heroku-20`, `heroku-22`, `heroku-24`, and `heroku-26`. The correct pre-built binary is selected automatically from the `$STACK` of the app being built.

## Vendored versions

Each stack ships its own pre-built tarball under `build/`. The `heroku-24` and `heroku-26` builds were refreshed to current upstream releases:

| Stack | ImageMagick | libheif | libde265 | libwebp |
| --- | --- | --- | --- | --- |
| `heroku-20` | 7.1.0-43 | 1.12.0 | 1.0.8 | 1.2.2 |
| `heroku-22` | 7.1.0-43 | 1.12.0 | 1.0.8 | 1.2.2 |
| `heroku-24` | 7.1.2-26 | 1.23.1 | 1.1.1 | 1.6.0 |
| `heroku-26` | 7.1.2-26 | 1.23.1 | 1.1.1 | 1.6.0 |

> **HEIC note:** On `heroku-26` the base image no longer provides an HEVC encoder, so HEIC *encoding* is unavailable on that stack (WebP and HEIC *decoding* work on all stacks). This has no effect on the typical WebP use case.

## Usage

Add this buildpack to your app:

```plain
heroku buildpacks:add https://github.com/speakerdeckllc/heroku-buildpack-imagemagick-webp -i 1 -a <app name>
```

And add it into your `app.json`:

```json
  "buildpacks": [
    {
      "url": "https://github.com/speakerdeckllc/heroku-buildpack-imagemagick-webp"
    },
    {
      "url": "heroku/ruby"
    }
  ],
```

## How it works

When you use this buildpack it unpacks the pre-built `build/imagemagick-heroku-$STACK.tar.gz` file into your Heroku application's `vendor/imagemagick` folder and sets up the relevant environment variables (`MAGICK_HOME`, `PATH`, `LD_LIBRARY_PATH`, etc.).

If you were to run a Heroku `bash` session you can investigate the dependencies:

```plain
$ heroku run -a <appname> bash

~ $ magick -version
Version: ImageMagick 7.1.2-26 Q16-HDRI x86_64 https://imagemagick.org
Delegates (built-in): bzlib djvu fontconfig freetype heic jbig jng jp2 jpeg lcms lqr lzma openexr png tiff webp x xml zip zlib zstd

~ $ dwebp -version
1.6.0
```

(HEIF support is built into the `magick` binary — see the `heic` delegate above. The standalone `heif-info`/`heif-enc` tools are not bundled.)

## Build script

The binaries are compiled inside the matching `heroku/heroku:<stack>-build` Docker image via `build.sh`, which takes the stack number as its argument:

```plain
./build.sh 24    # builds build/imagemagick-heroku-24.tar.gz
./build.sh 26    # builds build/imagemagick-heroku-26.tar.gz
```

Notes on the build:

- **Platform is pinned to `linux/amd64`.** Heroku dynos are x86_64, so `build.sh` always builds for that platform. On Apple Silicon this runs under emulation (slower, but produces correct binaries).
- **`heroku-24`/`heroku-26` images run as a non-root user**, so those Dockerfiles switch to `USER root` to install packages and write to `/usr/src`.
- **libde265 and libheif are built with CMake** (they dropped autotools support), with parallelism capped (`--parallel 2`) to avoid the compiler running out of memory in the Docker VM.

To update or add a stack:

1. Update or add the `Dockerfile.<stack>` (and add a branch in `bin/compile`).
2. Re-build the tarball:

    ```plain
    ./build.sh <stack>
    ```

3. Commit the changes, **including the `.tar.gz` file**, and push to your fork.
4. Purge your Heroku application's cache:

   ```plain
   heroku builds:cache:purge -a <app name>
   ```

5. Redeploy your application via the Heroku dashboard, or push a new commit.

### Credits

* <https://github.com/drnic/heroku-buildpack-imagemagick-webp>
* <https://github.com/brandoncc/heroku-buildpack-vips>
* <https://github.com/steeple-dev/heroku-buildpack-imagemagick>
* <https://github.com/slagkryssaren/heroku-buildpack-imagemagick-heif>
