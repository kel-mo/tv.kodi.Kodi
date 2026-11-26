# Kodi Flatpak

Kodi is an award-winning free and open source software media player and
entertainment hub for digital media. Available as a native application for
Android, Linux, BSD, macOS, iOS, tvOS and Windows operating systems, Kodi runs
on most common processor architectures. This repository packages it through
Flatpak.

## Installing

End-user instructions, including the beta channel, the optional BD-J extension
and where Kodi keeps its data, are on the
[Kodi wiki](https://kodi.wiki/view/HOW-TO:Install_Kodi_for_Linux#Flatpak).

## Building

Install the builder, then let it pull the runtime, SDK and SDK extensions the
manifest asks for (currently `org.freedesktop.Sdk` 26.08 and the OpenJDK 17
extension used by the BD-J extension):

```
flatpak install flathub org.flatpak.Builder
flatpak run org.flatpak.Builder --install-deps-from=flathub --install-deps-only build-dir tv.kodi.Kodi.yml
```

Build into the local `repo`, then add it as a remote and install from it. The
addon extensions are pulled from the same repo automatically. The flatpak
branch follows the git branch like Flathub does, `stable` from `master` and
`beta` from any branch containing `beta`; override with `BRANCH=`:

```
make build
make install
make run
```

`make uninstall` removes the local install and remote again; the app data in
`~/.var/app` is shared with any Flathub install of Kodi and is left alone.
Builder output is appended to `build.log`. When rerunning after fixing a
failed build, pass `BUILDER_FLAGS=--disable-download` to reuse the cached
sources rather than re-check them. To test on another machine, serve `repo`
over HTTP and use that URL with `flatpak remote-add`.

## Debugging

After `make install`, install the debug symbols from `repo` and start Kodi on
the Flatpak SDK with debug logging:

```
make debug
```

It runs Kodi with `--devel`, which swaps the Platform runtime for the SDK with
tools such as `gdb`, `strace` and `valgrind`, and `--debug`, which writes
Kodi's debug log to `~/.var/app/tv.kodi.Kodi/data/temp/kodi.log`.

For a Flathub install, do the same by hand (use `//beta` for the beta channel):

```
flatpak install flathub org.freedesktop.Sdk//26.08 tv.kodi.Kodi.Debug//stable
flatpak run --devel tv.kodi.Kodi//stable --debug
```

Attach `gdb` to the running Kodi. Get the instance ID from `flatpak ps`:

```
flatpak enter <instance-id> sh -c 'exec gdb -p $(pgrep kodi.bin)'
```

If Kodi crashes while `gdb` is available, the launcher writes a backtrace to
`~/.var/app/tv.kodi.Kodi/data/kodi_crashlog-<date>.log`.

## Debugging BD-J

The `tv.kodi.Kodi.bdj` extension ships libbluray's `bd_info`, which reports
whether BD-J was detected and whether a Java VM was found. Kodi's launcher sets
up the environment automatically, but other commands do not, so source the
extension's `env.sh` first:

```
flatpak run --command=sh tv.kodi.Kodi -c '. /app/share/kodi/extra/bdj/env.sh; bd_info /path/to/BDMV-or-iso'
```

## Maintenance

Preparing an update is a sequence of small commits, one per step, followed by
a local build and test:

```
make update-sources
make update-addons
make build && make install && make run
```

`update-sources` runs `flatpak-external-data-checker`, which edits the
manifest in place; install it once with
`flatpak install flathub org.flathub.flatpak-external-data-checker`.

`make lint-checkers` checks the update metadata offline. It needs a
`flatpak-external-data-checker` with `--lint` support.

`update-addons` updates the binary addons from `repo-binary-addons`, rewires
them as extensions and regenerates `addon-list.txt`. It needs a
`GITHUB_TOKEN` (exported or in `.env`). `uv run` installs the Python modules
it needs from the inline metadata in `tools/addon_updater.py`. To update only
some addons, name them, e.g.
`make update-addons UPDATE_ADDONS="inputstream.adaptive pvr.hts"`.

Kodi itself is only bumped by the checker when a release tag appears. While
the beta branch tracks Kodi master, update the `commit:` under the
`xbmc/xbmc.git` source by hand after `update-addons`, before building.

The `xbmc-master` branch tracks Kodi master and carries no patches that are
waiting to go upstream; it is never pushed to Flathub. `make update-sources`
moves its Kodi source to the newest commit on master, and it installs as the
`xbmc-master` Flatpak branch so it can sit beside `stable` and `beta`.

## Nightly builds

The nightly workflow, kept on its own in the `xbmc-master-nightly-ci` branch,
builds the `xbmc-master` branch with the newest Kodi every night around 02:00
Brisbane time, or when started from the Actions tab: it moves the Kodi source
to the newest commit on master, updates the addons in `addons-dev.txt` and
runs `make build-dev`. A good build is uploaded as an artifact holding the
Flatpak repo, kept for 7 days, and the pins it used are pushed to the
`xbmc-master-nightly` branch. A failed build publishes nothing.

To install one, download the `kodi-xbmc-master-<commit>-X64-repo` artifact
from the run (GitHub needs a login for artifact downloads), unpack it and
install from it. The runtime comes from Flathub, so the `flathub` remote has to
be there too. The addons in the dev build install with it:

```
gh run download <run-id> -R kel-mo/tv.kodi.Kodi -n kodi-xbmc-master-<commit>-X64-repo
tar -xf kodi-repo.tar
flatpak remote-add --user --no-gpg-verify kodi-nightly "file://$PWD/repo"
flatpak install --user kodi-nightly tv.kodi.Kodi//xbmc-master
```

Installing it makes it the Kodi that the desktop launcher starts. If another
Kodi Flatpak is installed, make that one current again with
`flatpak make-current --user tv.kodi.Kodi beta` (or `stable`).

Kodi master upgrades the databases of the profile it runs on, and the app data
in `~/.var/app/tv.kodi.Kodi` is shared with every other install of Kodi, so
start it with a profile of its own:

```
flatpak run --env=KODI_DATA=$HOME/.var/app/tv.kodi.Kodi/xbmc-master tv.kodi.Kodi//xbmc-master
```

For a newer nightly, replace `repo` with the one from its tar
(`rm -rf repo && tar -xf kodi-repo.tar`) and run
`flatpak update --user tv.kodi.Kodi//xbmc-master`.
`flatpak uninstall --user tv.kodi.Kodi//xbmc-master` and
`flatpak remote-delete --user kodi-nightly` remove it again; its profile stays
in `~/.var/app/tv.kodi.Kodi/xbmc-master`.

## Contributing

The list of binary addons in each branch of Kodi may be found
[here](https://github.com/xbmc/repo-binary-addons/), and dependencies
[here](https://github.com/xbmc/xbmc/tree/master/tools/depends/target). Kodi
releases are found [here](https://github.com/xbmc/xbmc/releases).

You can contribute by updating addons, modules and the Kodi version.
