# 09 — Install daily applications

## Goal

Install a compact, coherent set of graphical applications for the Niri daily
driver and make their most important file and URL associations reproducible
through `niri-dotfiles`.

This chapter:

- installs Firefox as the browser;
- installs Nautilus as the graphical file manager;
- installs Papers and Loupe for documents and images;
- installs GNOME Text Editor as a simple graphical editor while retaining
  Micro and Vim for terminal work;
- installs Celluloid for audio and video playback;
- installs File Roller and video-thumbnail support for Nautilus;
- installs GNOME Calculator, GNOME Calendar, and the LibreOffice maintenance
  branch;
- installs English and Spanish spell-checking dictionaries;
- adds Code - OSS, Vim, and GitHub CLI as the initial development tools;
- deploys a reviewed `mimeapps.list` from `niri-dotfiles`;
- verifies the applications without adding a launcher, theme, online account,
  or system daemon.

It does not install an email client, cloud client, password-manager UI, AUR
helper, Microsoft Visual Studio Code build, full programming-language
toolchain, game platform, image editor, or media-library manager. Calendar
account synchronization is also deferred. Those remain conditional choices
rather than silent additions to the workstation.

## Prerequisites

- Chapters 01 through 08 are complete.
- Niri starts manually and Kitty opens.
- The chapter 07 audio, removable-media, portal, and Secret Service checks
  pass.
- The standard XDG user directories from chapter 08 exist.
- The local `niri-dotfiles` clone is clean and contains the reviewed chapter
  09 `mimeapps` package.
- GNU Stow and the `xdg-mime` command are already installed.

Run ordinary commands as `neon` and use `sudo` only where shown. Application
tests in this chapter run inside Kitty under Niri, not from a bare TTY.

## Resulting application set

| Role | Canonical application | Reason for this baseline |
| --- | --- | --- |
| Web browser | Firefox | Official Arch package, mature Wayland-capable browser, and portal support. |
| File manager | Nautilus | Reuses the existing GVfs, UDisks, XDG-directory, portal, and GNOME integration. |
| PDF and document viewer | Papers | Current GNOME document viewer with PDF support and its own thumbnailer. |
| Image viewer | Loupe | Focused GTK 4 image viewer that fits the selected GNOME application stack. |
| Graphical text editor | GNOME Text Editor | Simple editor for ordinary text; it does not replace Micro, Vim, or Code. |
| Audio and video player | Celluloid | Mature GTK frontend for `mpv` without introducing a media-library service. |
| Archive interface | File Roller | Adds a graphical archive manager and a Nautilus extension over the chapter 08 tools. |
| Video thumbnails | ffmpegthumbnailer | Supplies previews for common local video files in the file manager. |
| Office suite | LibreOffice Still | Maintenance branch selected for a reliability-first daily driver. |
| Calculator | GNOME Calculator | Complete desktop calculator without a larger desktop-shell dependency. |
| Calendar | GNOME Calendar | Local calendar that fits the GTK stack; remote-account synchronization remains unconfigured. |
| Development editor | Code - OSS | Official-repository open-source VS Code build. |
| Terminal editors | Micro and Vim | Micro remains the convenient editor; Vim is present for the later learning guide. |
| GitHub client | GitHub CLI | Complements the existing Git and OpenSSH clients without enabling a daemon. |

The shared GTK and GNOME direction is intentional. Mixing one application
from every desktop environment would add overlapping settings, portals, and
integration layers without making this workstation more complete.

## Audit existing alternatives

Check whether an earlier test already installed an overlapping application:

```bash
pacman -Q firefox chromium nautilus thunar dolphin papers evince loupe eog gnome-text-editor gedit celluloid vlc file-roller gnome-calendar libreoffice-still libreoffice-fresh md4c code vim github-cli 2>&1
```

On the clean canonical path, most or all entries should be absent. A package
reported as installed is not automatically a problem, but stop before
installing a second application for the same role. Review its configuration
and decide whether to keep it, migrate from it, or deliberately maintain both.

In particular, `libreoffice-still` and `libreoffice-fresh` conflict and must
not be installed together. This profile selects `libreoffice-still`, the
maintenance branch. Do not remove an existing office suite until its templates,
dictionaries, extensions, and user documents have been accounted for.

Inspect any existing user MIME configuration:

```bash
ls -l ~/.config/mimeapps.list
```

`No such file or directory` is expected on a clean system. If the file exists,
read it before continuing:

```bash
cat ~/.config/mimeapps.list
```

Do not let Stow overwrite it. The deployment section describes the conflict
boundary.

## Install the canonical applications

Read new entries on the [Arch Linux news page](https://archlinux.org/news/),
then perform one complete upgrade transaction:

```bash
sudo pacman -Syu \
    firefox \
    nautilus \
    papers \
    loupe \
    gnome-text-editor \
    celluloid \
    file-roller \
    ffmpegthumbnailer \
    gnome-calculator \
    gnome-calendar \
    libreoffice-still \
    md4c \
    hunspell-en_us \
    hunspell-es_es \
    code \
    vim \
    github-cli
```

Read the complete transaction before accepting it. Nautilus deliberately
pulls the GNOME search and metadata stack; Celluloid pulls `mpv`; GNOME
Calendar pulls Evolution Data Server; LibreOffice is the largest application
in this chapter; and `md4c` provides the `libmd4c.so.0` runtime library loaded
by the current Writer component. Listing that small library package explicitly
keeps a clean installation functional even if the LibreOffice Still dependency
metadata does not pull it in automatically. These are known parts of the
selected functions, not reasons to accept unrelated optional packages.

The dictionaries provide spelling data for US English and Spanish from Spain.
They do not change `LANG`, the physical keyboard layout, Firefox's interface
language, or LibreOffice's interface language. The canonical interface remains
English.

Inspect unresolved package configuration files:

```bash
sudo pacdiff --output
```

Resolve every reported file before proceeding.

## Understand the initial development boundary

The `code` package in the official repositories is **Code - OSS**, not the
Microsoft-branded Visual Studio Code binary. It launches with:

```bash
code
```

Its desktop entry is `code-oss.desktop`. Microsoft account integration and
the Microsoft extension Marketplace are not assumed by this runbook. Do not
add unofficial Marketplace endpoints, install an AUR binary, or replace the
package merely to make a copied extension command work. The handbook will
compare those choices before the development environment is expanded.

Vim is installed without plugins or a user configuration. Micro remains
available, and GNOME Text Editor handles ordinary graphical text files. Their
roles can coexist:

```bash
micro --version
vim --version | head -n 3
code --version
gh --version
```

Installing GitHub CLI does not authenticate it. Do not copy another machine's
`~/.config/gh/hosts.yml`, paste a token into the repository, or enable SSH
server access. GitHub authentication and Secret Service storage belong in the
handbook guide; `sshd.service` remains disabled.

### Conditional: Python development tools

LibreOffice depends on the system Python interpreter, but that incidental
dependency is not a complete project environment. If this ThinkPad will run
Python projects, explicitly install the supported package and environment
tools:

```bash
sudo pacman -Syu python python-pip python-pipx
```

Use one virtual environment per project and `pipx` for standalone Python
applications. Do not use `sudo pip`, and do not override Arch's externally
managed system environment. PySide or PyQt, Qt Designer, database connectors,
PlatformIO, and embedded toolchains must be selected per project in the
handbook.

### Conditional: building Arch packages

Install the build-tool group only when a reviewed PKGBUILD or local build
actually requires it:

```bash
sudo pacman -Syu base-devel
```

`base-devel` is required for normal Arch package building, but installing it
does not make AUR content trusted and does not install an AUR helper. The
chapter 02 AUR policy still applies.

## Verify commands and desktop entries

Confirm that every canonical command resolves:

```bash
command -v firefox nautilus papers loupe gnome-text-editor celluloid file-roller ffmpegthumbnailer gnome-calculator gnome-calendar libreoffice code vim gh
```

Every name must print one executable path. Confirm that the desktop files used
by the tracked MIME configuration exist:

```bash
ls -l \
    /usr/share/applications/firefox.desktop \
    /usr/share/applications/org.gnome.Nautilus.desktop \
    /usr/share/applications/org.gnome.Papers.desktop \
    /usr/share/applications/org.gnome.Loupe.desktop \
    /usr/share/applications/org.gnome.TextEditor.desktop \
    /usr/share/applications/io.github.celluloid_player.Celluloid.desktop \
    /usr/share/applications/org.gnome.FileRoller.desktop \
    /usr/share/applications/org.gnome.Calendar.desktop \
    /usr/share/applications/libreoffice-writer.desktop \
    /usr/share/applications/libreoffice-calc.desktop \
    /usr/share/applications/libreoffice-impress.desktop \
    /usr/share/applications/code-oss.desktop
```

All paths must exist. These desktop-file IDs, not the display names visible in
a future launcher, are the values recorded in `mimeapps.list`.

Confirm the installed package set:

```bash
pacman -Q \
    firefox \
    nautilus \
    papers \
    loupe \
    gnome-text-editor \
    celluloid \
    file-roller \
    ffmpegthumbnailer \
    gnome-calculator \
    gnome-calendar \
    libreoffice-still \
    md4c \
    hunspell-en_us \
    hunspell-es_es \
    code \
    vim \
    github-cli
```

Confirm that the Markdown parser library belongs to the installed `md4c`
package and that Writer has no unresolved shared-library dependency:

```bash
pacman -Qo /usr/lib/libmd4c.so.0
ldd /usr/lib/libreoffice/program/libswlo.so | grep 'not found'
```

The first command must identify `md4c`. The second command must print nothing;
in this check, no output means that every required shared library was found.
If it instead reports `libmd4c.so.0 => not found`, stop before testing Writer,
install `md4c` through a complete upgrade transaction, and repeat both checks:

```bash
sudo pacman -Syu md4c
```

Do not install `base-devel`, `fakeroot`, or an AUR helper to repair this error.
They do not provide `libmd4c.so.0`.

## Deploy the default-application map

The chapter 09 `niri-dotfiles` update adds this Stow package:

```text
mimeapps/.config/mimeapps.list
```

It owns only portable user defaults. It does not store recent files, browser
profiles, application state, credentials, or downloaded content.

Go to the repository clone and inspect the tracked file:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
cat mimeapps/.config/mimeapps.list
```

The Git tree must be clean. The file must contain a `[Default Applications]`
section and an `[Added Associations]` entry that associates `inode/directory`
with `org.gnome.Nautilus.desktop`. Declaring a default alone is insufficient
when an implementation cannot otherwise confirm that the desktop file handles
that MIME type. Preview the deployment:

```bash
stow --simulate --verbose --no-folding --target="$HOME" mimeapps
```

If `~/.config/mimeapps.list` did not exist, the preview should show one new
link and no conflict. Deploy it:

```bash
stow --verbose --no-folding --target="$HOME" mimeapps
```

If an untracked `~/.config/mimeapps.list` already existed, stop at the preview.
Compare it with the repository version and preserve it under a deliberate
backup name before deploying. For example, after confirming that the backup
target does not already exist:

```bash
test ! -e ~/.config/mimeapps.list.before-chapter-09
mv ~/.config/mimeapps.list ~/.config/mimeapps.list.before-chapter-09
stow --verbose --no-folding --target="$HOME" mimeapps
```

Do not run the `mv` command when the source is absent, and do not overwrite an
older backup. Merge any intentional custom association into the tracked file
instead of discarding it.

Confirm ownership and content:

```bash
ls -l ~/.config/mimeapps.list
readlink -f ~/.config/mimeapps.list
cat ~/.config/mimeapps.list
```

`readlink -f` must resolve below the local `niri-dotfiles/mimeapps` directory.

## Verify default associations

Query the high-value defaults:

```bash
xdg-mime query default x-scheme-handler/https
xdg-mime query default inode/directory
xdg-mime query default application/pdf
xdg-mime query default image/jpeg
xdg-mime query default text/plain
xdg-mime query default video/mp4
xdg-mime query default application/zip
xdg-mime query default text/calendar
xdg-mime query default application/vnd.oasis.opendocument.text
```

The expected values, in order, are:

```text
firefox.desktop
org.gnome.Nautilus.desktop
org.gnome.Papers.desktop
org.gnome.Loupe.desktop
org.gnome.TextEditor.desktop
io.github.celluloid_player.Celluloid.desktop
org.gnome.FileRoller.desktop
org.gnome.Calendar.desktop
libreoffice-writer.desktop
```

The explicit `[Added Associations]` entry for `inode/directory` matters here.
Kitty installs `kitty-open.desktop` as a valid directory handler, so it can be
selected when Nautilus is named as the default but is not also recognized as
associated with directories.

Inspect the registered handlers as a second check:

```bash
gio mime inode/directory
gio mime application/pdf
gio mime image/jpeg
gio mime video/mp4
gio mime text/calendar
```

Each result must identify the same default application. If the directory query
still returns `kitty-open.desktop`, do not remove or modify Kitty. Confirm the
deployed declarations and register Nautilus with the standard user command:

```bash
grep -n -E '^\[Default Applications\]|^\[Added Associations\]|^inode/directory=' \
    ~/.config/mimeapps.list
xdg-mime default org.gnome.Nautilus.desktop inode/directory
xdg-mime query default inode/directory
gio mime inode/directory
```

The last two commands must now identify `org.gnome.Nautilus.desktop`. Because
the deployed file is a Stow symlink, inspect `git diff` in `niri-dotfiles`
after running the repair and retain only the intended association. If a query
is empty, confirm that the desktop file exists, that the Stow link resolves,
and that the MIME name is spelled exactly as shown before changing anything.

Do not use `sudo xdg-mime`: defaults belong to `neon`. With the Stow package
deployed, a graphical application's **Make default** button or an
`xdg-mime default` command may edit the tracked file through its symlink.
Review `git status` after such a change and commit only intentional defaults.

## Perform functional tests under Niri

Open the Arch website and the Downloads directory through the generic desktop
interfaces:

```bash
xdg-open https://archlinux.org/
xdg-open "$(xdg-user-dir DOWNLOAD)"
```

Firefox and Nautilus must open. In Firefox, visit `about:support` and confirm
that **Window Protocol** reports Wayland. Use a page with a file-upload control
to confirm that the portal file chooser opens. Nautilus must show the standard
XDG directories and any removable volume mounted by udiskie.

Launch the remaining applications from Kitty while chapter 10 has not yet
installed a launcher:

```bash
papers
loupe
gnome-text-editor
celluloid
file-roller
gnome-calculator
gnome-calendar
libreoffice --writer
code
```

Use representative, non-sensitive local files to test the viewers:

- open one PDF in Papers and confirm page rendering and search;
- open JPEG, PNG, and WebP images in Loupe;
- play one local audio file and one local video file in Celluloid;
- open a ZIP archive in File Roller and inspect it before extracting;
- create one disposable local event in GNOME Calendar, close the application,
  reopen it, and confirm that the event remains;
- create a disposable Writer document, close it, and reopen it;
- open a small existing Git repository in Code - OSS without installing an
  extension bundle yet.

The applications should appear in the Niri overview even though no launcher
has been installed. Do not add temporary Niri key bindings for every program;
chapter 10 will supply the normal application-launch path.

## Check spelling and document integration

In Firefox, right-click in a multiline text field and confirm that US English
and Spanish dictionaries can be selected. In LibreOffice Writer, inspect the
language shown for the current paragraph and type a short English and Spanish
sample.

The dictionaries assist applications that use Hunspell. They do not translate
the interface and they do not force one document language globally. Select the
language appropriate to the current text instead of disabling spell checking
when a document changes language.

If the optional printer path from chapter 07 was skipped, LibreOffice may list
only file or PDF output. Do not enable CUPS merely to satisfy a print-dialog
test when no printer is required.

## Understand the calendar backend

GNOME Calendar stores and retrieves calendar data through Evolution Data
Server. The dependency provides per-user source-registry, calendar-factory,
and alarm-notification services. The source registry and factory are
activation-based user services. The alarm notifier is different: it must keep
running in the user session so it can notice a reminder while GNOME Calendar
is closed. Do not enable any of them as system services and do not add a
`spawn-at-startup` command for them to the Niri configuration.

After opening GNOME Calendar, inspect the installed user units:

```bash
systemctl --user list-unit-files 'evolution-*' --no-pager
systemctl --user status evolution-source-registry.service evolution-calendar-factory.service --no-pager
```

The factory units are expected to be static and may be active after the
application requests them. Static does not mean broken: D-Bus activates them
when a client needs them.

Evolution Data Server also installs an XDG autostart entry and an installable
systemd user unit for the alarm notifier. Inspect both:

```bash
ls -l /etc/xdg/autostart/org.gnome.Evolution-alarm-notify.desktop
systemctl --user list-unit-files evolution-alarm-notify.service --no-pager
systemctl --user status evolution-alarm-notify.service --no-pager
```

On a clean Niri installation, the unit may report `disabled` under **STATE**
and `enabled` under **PRESET**. These columns answer different questions:
`STATE` describes the actual per-user enablement, whereas `PRESET` describes
the package vendor's recommended default. A reboot does not change a disabled
unit merely because its preset is enabled.

Enable and start the alarm notifier explicitly for `neon`:

```bash
systemctl --user enable --now evolution-alarm-notify.service
systemctl --user is-enabled evolution-alarm-notify.service
systemctl --user is-active evolution-alarm-notify.service
systemctl --user status evolution-alarm-notify.service --no-pager
```

The two short checks must return `enabled` and `active`. Do not use `sudo`:
this is a user service. Enabling the packaged unit is preferable to editing
the vendor XDG desktop file or starting a second copy from Niri. Chapter 10
will install the notification daemon and then test a short disposable reminder
end to end.

This chapter creates only a local calendar. Do not install GNOME Control
Center solely for its online-account panel, enter Google credentials, or add a
CalDAV URL yet. Remote synchronization changes the credential, network,
conflict-resolution, and backup model and will be documented separately.
Calendar databases, caches, account sources, and secrets are generated user
state and must not be added to `niri-dotfiles`.

## Verify the completed checkpoint

Confirm that no system service was enabled by this chapter, the intended user
alarm service is active, and no unit failed:

```bash
systemctl --user is-enabled evolution-alarm-notify.service
systemctl --user is-active evolution-alarm-notify.service
systemctl --failed --no-pager
systemctl --user --failed --no-pager
sudo ss -lntup
```

The application packages require no new inbound firewall rule. Firefox, Code,
and other clients make outbound connections while in use; that is different
from enabling a listening server. The local calendar backend uses per-user
D-Bus and does not require a listening network port. Investigate any new
listener rather than opening a port for it.

Confirm that remote SSH login remains disabled:

```bash
systemctl is-enabled sshd.service 2>&1
systemctl is-active sshd.service 2>&1
```

Both results should remain `disabled` and `inactive` unless remote access was
separately designed and secured outside this chapter.

Finally, verify the tracked user configuration is unchanged after the tests:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
git status --short --branch
```

The tree should be clean. If an application changed `mimeapps.list`, inspect
the diff and either keep a deliberate change or restore the reviewed default
map through Git. Do not commit unrelated application state.

## Optional applications deliberately omitted

Install these only after a real requirement is identified:

| Requirement | Candidate | Why it is not canonical yet |
| --- | --- | --- |
| Raster image editing | GIMP | Loupe already covers viewing; an editor adds a different workflow and configuration surface. |
| Email client | Thunderbird | No account or local-mail policy has been selected. |
| Dedicated music library | A maintained music player | Celluloid handles local playback; library indexing and metadata policy remain undecided. |
| Microsoft-branded VS Code | Upstream or AUR package | It leaves the official-repository-only baseline and changes telemetry, Marketplace, and update decisions. |
| Proprietary media codecs or DRM extras | Application-specific component | Install only for a demonstrated format or service requirement. |

When one becomes necessary, document its source, update mechanism, secrets,
file associations, Wayland behavior, and recovery path rather than appending it
silently to the main package command.

## Recovery and later changes

To stop calendar reminders without deleting local events or disabling the
calendar backend:

```bash
systemctl --user disable --now evolution-alarm-notify.service
```

Restore the canonical reminder behavior with:

```bash
systemctl --user enable --now evolution-alarm-notify.service
```

Neither command needs `sudo`, and neither changes the static D-Bus-activated
source-registry or calendar-factory units.

To remove only the deployed default map while preserving the repository file:

```bash
cd ~/Projects/CycloniteRDX/niri-dotfiles
stow --delete --verbose --target="$HOME" mimeapps
```

If this chapter moved an older map to
`~/.config/mimeapps.list.before-chapter-09`, restore it only after the Stow
link has been removed and the destination is absent:

```bash
test ! -e ~/.config/mimeapps.list
mv ~/.config/mimeapps.list.before-chapter-09 ~/.config/mimeapps.list
```

To change a default reproducibly, edit
`mimeapps/.config/mimeapps.list` in the repository, review the Git diff, and
reconcile the package:

```bash
stow --restow --verbose --no-folding --target="$HOME" mimeapps
```

Before removing a canonical application, identify packages that depend on it:

```bash
pactree --reverse package-name
```

Replace `package-name`; never paste the placeholder literally. Review a
precise `pacman -Rns` transaction before confirming it. Removing Nautilus,
LibreOffice, or another selected package can also remove integrations or leave
its MIME default pointing to a missing desktop file, so update the tracked map
in the same reviewed change.

## Completion checklist

- [ ] Firefox opens HTTPS links and reports Wayland in `about:support`.
- [ ] Nautilus opens directories and sees the standard XDG locations.
- [ ] Papers opens PDFs and provides document thumbnails.
- [ ] Loupe opens common image formats.
- [ ] Celluloid plays local audio and video through PipeWire.
- [ ] File Roller opens ZIP and 7-Zip archives and integrates with Nautilus.
- [ ] LibreOffice Still opens Writer, Calc, and Impress documents.
- [ ] `libmd4c.so.0` belongs to `md4c`, and Writer reports no missing shared
      library.
- [ ] GNOME Calendar preserves a disposable local event after restarting.
- [ ] Evolution Data Server factory units remain D-Bus-activated and were not
      manually enabled as system services.
- [ ] `evolution-alarm-notify.service` is enabled and active as a user service.
- [ ] No Google, CalDAV, or other online calendar account has been configured.
- [ ] US English and Spanish spell checking are available.
- [ ] GNOME Text Editor is the ordinary graphical text-file default.
- [ ] Micro and unconfigured Vim remain available in Kitty.
- [ ] Code - OSS and GitHub CLI run without copying credentials or enabling a
      server.
- [ ] Optional Python or package-building tools were installed only if this
      machine needs them.
- [ ] `~/.config/mimeapps.list` resolves to the reviewed Stow package.
- [ ] URL, directory, PDF, image, text, media, archive, and office defaults
      match the documented desktop-file IDs.
- [ ] No failed unit, unexplained listener, or firewall exception was added.
- [ ] `sshd.service` remains disabled and inactive.
- [ ] Manual Niri startup, Kitty, and TTY recovery still work.

## Sources

- [ArchWiki: General recommendations](https://wiki.archlinux.org/title/General_recommendations)
- [ArchWiki: List of applications](https://wiki.archlinux.org/title/List_of_applications)
- [ArchWiki: XDG MIME Applications](https://wiki.archlinux.org/title/XDG_MIME_Applications)
- [ArchWiki: Firefox](https://wiki.archlinux.org/title/Firefox)
- [ArchWiki: LibreOffice](https://wiki.archlinux.org/title/LibreOffice)
- [ArchWiki: Visual Studio Code](https://wiki.archlinux.org/title/Visual_Studio_Code)
- [ArchWiki: Python](https://wiki.archlinux.org/title/Python)
- [systemctl(1)](https://man.archlinux.org/man/systemctl.1)
- [systemd.preset(5)](https://man.archlinux.org/man/systemd.preset.5)
- [Arch packages: Firefox](https://archlinux.org/packages/extra/x86_64/firefox/)
- [Arch packages: Nautilus](https://archlinux.org/packages/extra/x86_64/nautilus/)
- [Arch packages: Papers](https://archlinux.org/packages/extra/x86_64/papers/)
- [Arch packages: Loupe](https://archlinux.org/packages/extra/x86_64/loupe/)
- [Arch packages: Celluloid](https://archlinux.org/packages/extra/x86_64/celluloid/)
- [Arch packages: File Roller](https://archlinux.org/packages/extra/x86_64/file-roller/)
- [Arch packages: GNOME Calendar](https://archlinux.org/packages/extra/x86_64/gnome-calendar/)
- [Arch packages: Evolution Data Server](https://archlinux.org/packages/extra/x86_64/evolution-data-server/)
- [Arch packages: LibreOffice Still](https://archlinux.org/packages/extra/x86_64/libreoffice-still/)
- [Arch packages: md4c file list](https://archlinux.org/packages/extra/x86_64/md4c/files/)
- [Arch packages: Code - OSS](https://archlinux.org/packages/extra/x86_64/code/)
- [Arch packages: GitHub CLI](https://archlinux.org/packages/extra/x86_64/github-cli/)

## Next step

Continue with chapter 10 to select and integrate the application launcher,
status UI, notification daemon, screenshot tooling, wallpaper manager, theme,
and logout interface. Those components will turn this tested application set
into a convenient daily desktop without hiding the TTY recovery path.
