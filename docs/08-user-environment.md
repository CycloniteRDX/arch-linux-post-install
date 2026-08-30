# 08 — Build the user environment

## Goal

Create the standard per-user directories and add a compact set of tools that
graphical applications, documentation work, and ordinary terminal sessions
need. Preserve the English system language and the machine-specific physical
keyboard selection made by the installation runbook.

This chapter:

- creates the standard XDG user directories for `neon`;
- installs general Latin-script fonts, color emoji, and metric-compatible
  document fonts;
- keeps Bash as the default shell and adds its packaged completions;
- adds native Wayland clipboard commands used by Micro and other tools;
- installs ZIP and 7-Zip support in addition to the archive formats already
  covered by the base system;
- adds a small, named set of search, inspection, JSON, copy, and monitoring
  tools;
- installs the freedesktop.org helpers used to open files and validate desktop
  entries.

It does not customize Bash, Kitty, Micro, or Vim; install a clipboard-history
daemon; select default graphical applications; install icon fonts; or replace
standard commands with aliases. Those choices belong to later dotfiles,
desktop-component, application, or handbook work.

## Prerequisites

- Chapters 01 through 07 are complete.
- Niri still starts manually and Kitty opens.
- The current graphical session belongs to `neon`.
- `LANG=en_US.UTF-8` is the canonical system language.
- `/etc/vconsole.conf` contains `KEYMAP=us` or `KEYMAP=es`, matching the
  physical keyboard on this ThinkPad.
- `micro`, `man-db`, `man-pages`, Git, and OpenSSH are already installed by the
  runbook or earlier post-install chapters.

Run ordinary commands as `neon` and use `sudo` only where shown.

## Resulting baseline

| Area | Canonical result |
| --- | --- |
| User directories | English-named Desktop, Documents, Downloads, Music, Pictures, Public, Templates, and Videos |
| System language | `en_US.UTF-8` |
| Physical keyboard | US or Spanish, selected independently for each machine |
| Default shell | Bash |
| Shell completion | Distribution-provided `bash-completion` |
| General fonts | DejaVu from chapter 05 plus Noto |
| Emoji | Noto Color Emoji |
| Document compatibility | Liberation Sans, Serif, and Mono |
| Wayland clipboard CLI | `wl-copy` and `wl-paste` |
| Archives | Base `tar`, gzip, bzip2, xz, and zstd support plus ZIP and 7-Zip |
| Search and selection | `rg`, `fd`, and `fzf` |
| Inspection and data | `bat`, `tree`, `jq`, `btop`, and `lsof` |
| File transfer | `rsync` |
| Desktop helpers | `xdg-open`, `xdg-mime`, `mimeopen`, and `desktop-file-validate` |

## Confirm language and keyboard independence

Inspect the effective session locale and the system selections:

```bash
locale
localectl
cat /etc/locale.conf
cat /etc/vconsole.conf
```

The session and `/etc/locale.conf` must use `en_US.UTF-8`. The console keymap
may be `us` on one ThinkPad and `es` on the other. A Spanish physical keyboard
does not require Spanish system messages, and changing the keymap does not
translate directory names.

The runbook already documents `es_ES.UTF-8` as an optional system-language
alternative and `KEYMAP=es` as a separate keyboard choice. Do not set
`LC_ALL` globally and do not change either value in this chapter.

## Audit the existing baseline

Confirm the packages inherited from the installation and graphical bootstrap:

```bash
pacman -Q \
    ttf-dejavu \
    micro \
    man-db \
    man-pages \
    git \
    openssh
```

Confirm the current login shell:

```bash
getent passwd "$USER" | cut -d: -f1,7
printf '%s\n' "$SHELL"
```

Both results should identify `/bin/bash`. Do not run `chsh`: this profile keeps
Bash as the predictable login and recovery shell. Fish, Zsh, Nushell, prompt
frameworks, aliases, and shell plugins can be evaluated later without changing
the system baseline.

## Install the user-environment packages

Read new entries on the [Arch Linux news page](https://archlinux.org/news/),
then perform one complete upgrade transaction:

```bash
sudo pacman -Syu \
    xdg-user-dirs \
    xdg-utils \
    perl-file-mimeinfo \
    desktop-file-utils \
    noto-fonts \
    noto-fonts-emoji \
    ttf-liberation \
    bash-completion \
    texinfo \
    wl-clipboard \
    zip \
    unzip \
    7zip \
    ripgrep \
    fd \
    fzf \
    jq \
    tree \
    bat \
    btop \
    rsync \
    lsof
```

Read the transaction before accepting it. The package is named `7zip`, not
the superseded `p7zip`; it still provides the familiar `7z` command.

The installed packages have explicit responsibilities:

| Package group | Purpose |
| --- | --- |
| `xdg-user-dirs` | Create and publish the standard user-directory paths. |
| `xdg-utils`, `perl-file-mimeinfo` | Let applications open URLs and files through registered desktop defaults. |
| `desktop-file-utils` | Validate desktop entries and maintain the application MIME cache. |
| `noto-fonts`, `noto-fonts-emoji` | Supply broad general-script coverage and color emoji. |
| `ttf-liberation` | Supply free metric-compatible substitutes for Arial, Times New Roman, and Courier New. |
| `bash-completion`, `texinfo` | Add command-specific completion and GNU Info documentation. |
| `wl-clipboard` | Provide native Wayland clipboard commands and Micro's optional Wayland clipboard backend. |
| `zip`, `unzip`, `7zip` | Create and extract common cross-platform archives. |
| `ripgrep`, `fd`, `fzf` | Search text, find paths, and select interactively. |
| `jq`, `tree`, `bat` | Process JSON and inspect directory or text content. |
| `btop`, `lsof` | Inspect resource use and open files or sockets. |
| `rsync` | Copy local or remote file trees efficiently; it is not a backup policy by itself. |

Inspect unresolved package configuration files:

```bash
sudo pacdiff --output
```

Resolve every reported file before proceeding.

## Create the XDG user directories

Run the update as `neon`, without `sudo`:

```bash
xdg-user-dirs-update
```

Because the session language is English, the first run should create:

```text
~/Desktop
~/Documents
~/Downloads
~/Music
~/Pictures
~/Public
~/Templates
~/Videos
```

Read the generated mapping and its recorded locale:

```bash
cat ~/.config/user-dirs.dirs
cat ~/.config/user-dirs.locale
```

Then confirm that the directories exist:

```bash
ls -ld \
    ~/Desktop \
    ~/Documents \
    ~/Downloads \
    ~/Music \
    ~/Pictures \
    ~/Public \
    ~/Templates \
    ~/Videos
```

Applications should query these paths instead of assuming literal directory
names. For example:

```bash
xdg-user-dir DOWNLOAD
xdg-user-dir DOCUMENTS
xdg-user-dir PICTURES
```

The results should point below `/home/neon`. `Public` is only a directory name;
creating it does not enable a web server, network share, discovery service, or
firewall exception.

Do not run `xdg-user-dirs-update --force` after storing files in these
directories merely to translate or rename them. A locale migration must first
account for existing files and application references. Detailed customization
and migration belong in the handbook.

## Activate Bash completion

Arch's `/etc/bash.bashrc` automatically sources the completion framework when
the package is installed. Open a new Kitty window or tab so it starts a new
Bash process, then run:

```bash
type _completion_loader
```

Bash should report that `_completion_loader` is a function. Type part of a
supported command and press Tab twice; for example:

```text
systemctl sta<Tab><Tab>
```

Do not add another unconditional `source` line to `~/.bashrc`; that would load
the same framework twice. This chapter also leaves the optional `fzf` key
bindings and completion scripts disabled until the shell dotfiles are designed.

## Verify fonts and fallback

Pacman's font hooks update the system Fontconfig cache during installation.
Check the selected families:

```bash
fc-match sans-serif
fc-match serif
fc-match monospace
fc-match 'Noto Color Emoji'
```

The first three commands should resolve to an installed Noto, Liberation, or
DejaVu font; the last must resolve to `NotoColorEmoji.ttf`.

Display a compact glyph test in Kitty:

```bash
printf '%s\n' 'English — Español: áéíóú ñ ¿¡ — € → ✓ 😀'
```

Ordinary Latin text, punctuation, the euro sign, arrows, the check mark, and
the emoji should render without replacement boxes. Do not add a user
`fonts.conf` merely because a different family was preferred visually; final
font selection and Kitty configuration belong in `niri-dotfiles`.

Nerd Fonts symbols are intentionally deferred until chapter 10 selects the
bar, launcher, and status icons. Installing an icon font now would make an
unselected desktop theme part of the system baseline.

### Optional: add CJK coverage

Skip this package when the workstation only needs the canonical English and
Spanish coverage. To display Chinese, Japanese, and Korean text comprehensively,
install Noto CJK:

```bash
sudo pacman -Syu noto-fonts-cjk
```

This is an official package but is much larger than the other font packages:
approximately 300 MiB installed at the time this chapter was written. Its
absence does not affect a Spanish physical keyboard or Spanish-language text.

## Verify the native Wayland clipboard

Run the test inside Kitty under Niri, not from a bare TTY:

```bash
printf '%s' 'Wayland clipboard test' | wl-copy
wl-paste
wl-copy --clear
```

`wl-paste` must print `Wayland clipboard test`. The final command removes the
disposable value.

The CLIPBOARD and PRIMARY selections are separate. PRIMARY is the traditional
selection commonly associated with selecting text and middle-clicking:

```bash
printf '%s' 'Primary selection test' | wl-copy --primary
wl-paste --primary
wl-copy --primary --clear
```

Do not start clipboard persistence or history yet. Clipboard managers retain
copied material, which can include passwords, tokens, private URLs, or other
sensitive text. Chapter 10 will select and configure one only after deciding
its retention and exclusion policy.

Installing `wl-clipboard` gives Micro its supported Wayland copy-and-paste
backend. It does not change Vim's compiled features or configure Vim
registers; the Vim and clipboard-provider guide belongs in the handbook.

## Use archive tools conservatively

The base system already contains `tar`, gzip, bzip2, xz, and zstd support.
This chapter adds commands needed for common Windows and downloaded archives:

| Task | Command pattern |
| --- | --- |
| List a tar archive | `tar -tf archive.tar.zst` |
| Extract a tar archive | `tar -xaf archive.tar.zst` |
| Create a compressed tar archive | `tar -caf archive.tar.zst directory/` |
| List a ZIP archive | `unzip -l archive.zip` |
| Extract a ZIP archive | `unzip archive.zip -d destination/` |
| Create a ZIP archive | `zip -r archive.zip directory/` |
| Test a 7-Zip or RAR archive | `7z t archive.7z` |
| Extract through 7-Zip | `7z x archive.7z -odestination` |
| Create a 7-Zip archive | `7z a archive.7z directory/` |

Before extracting an untrusted archive, list or test it and use a new empty
destination owned by `neon`. Do not extract downloads with `sudo`. Archives can
contain misleading paths, symlinks, executable files, or content designed to
overwrite an existing name. Detailed format, encryption, and recovery guidance
belongs in the handbook.

## Learn the installed console-tool boundary

These tools supplement rather than replace the standard commands:

| Tool | First useful command | Boundary |
| --- | --- | --- |
| ripgrep | `rg 'pattern' path/` | Respects ignore files by default; use standard `grep` when portability matters. |
| fd | `fd 'name' path/` | Friendly path search; scripts may still require POSIX `find`. |
| fzf | `fzf` | Reads candidate lines interactively; no shell key bindings are enabled yet. |
| jq | `jq . file.json` | Parses JSON; do not use text substitution as a JSON parser. |
| tree | `tree -L 2 path/` | Produces a compact hierarchy without changing files. |
| bat | `bat file` | Adds highlighting and paging; `cat` remains unchanged. |
| btop | `btop` | Interactive resource view; it is not a benchmark. |
| rsync | `rsync -a --dry-run source/ destination/` | Review without writing first; trailing slashes change copy semantics. |
| lsof | `sudo lsof -i -P -n` | Inspect open network endpoints; elevated output can expose process details. |

Do not add aliases such as `cat=bat`, `find=fd`, or `grep=rg` in this chapter.
The programs have different options and behavior, so replacing standard names
can make copied commands and scripts surprising.

## Verify the completed checkpoint

Confirm that the key executables resolve:

```bash
command -v \
    xdg-user-dirs-update \
    xdg-user-dir \
    xdg-open \
    xdg-mime \
    mimeopen \
    desktop-file-validate \
    wl-copy \
    wl-paste \
    zip \
    unzip \
    7z \
    rg \
    fd \
    fzf \
    jq \
    tree \
    bat \
    btop \
    rsync \
    lsof
```

Every name must print one executable path.

Validate the chapter 07 autostart desktop entry:

```bash
desktop-file-validate ~/.config/autostart/udiskie.desktop
```

No output means the file passes validation. Do not treat an empty result from
the following queries as an error yet:

```bash
xdg-mime query default inode/directory
xdg-mime query default application/pdf
```

Chapter 09 has not selected the graphical file manager or PDF reader, so those
default associations may legitimately be unset.

Confirm that the new packages did not introduce a failed service or network
listener:

```bash
systemctl --failed --no-pager
systemctl --user --failed --no-pager
sudo ss -lntup
```

This chapter enables no system daemon and opens no firewall port. Investigate
any new listener rather than attributing it to the console tools.

## Recovery and later changes

The package installation does not alter the boot chain or enable a daemon. If
one optional tool proves unwanted, identify its reverse dependencies before
removing that exact package:

```bash
pactree --reverse package-name
sudo pacman -Rns package-name
```

Replace `package-name`; never paste the placeholder literally. Review pacman's
complete removal transaction before confirming it.

Do not delete an XDG directory to disable it when it contains user files.
Applications read the paths from `~/.config/user-dirs.dirs`; safe renaming and
migration require moving the data and updating the mapping together. Preserve
the generated file until that process is documented in the handbook.

## Completion checklist

- [ ] The session language and `/etc/locale.conf` use `en_US.UTF-8`.
- [ ] The console keymap remains `us` or `es` for the physical keyboard.
- [ ] Bash remains the login shell and packaged completion loads in a new
      shell.
- [ ] All eight XDG user directories exist below `/home/neon`.
- [ ] `~/.config/user-dirs.dirs` records the expected English paths.
- [ ] Noto, Noto Color Emoji, Liberation, and DejaVu resolve through
      Fontconfig.
- [ ] The Kitty glyph test renders English, Spanish, symbols, and emoji.
- [ ] CJK support is either deliberately absent or installed as the documented
      optional package.
- [ ] `wl-copy` and `wl-paste` pass both clipboard tests under Niri.
- [ ] No clipboard-history or persistence daemon has been added yet.
- [ ] ZIP and 7-Zip commands are available alongside the base tar tools.
- [ ] The selected console utilities resolve without replacing standard
      command names.
- [ ] The udiskie desktop entry passes `desktop-file-validate`.
- [ ] Graphical MIME defaults are allowed to remain unset until chapter 09.
- [ ] No failed unit, unexpected network listener, service, or firewall rule
      was introduced.
- [ ] Manual Niri startup and TTY recovery still work.

## Sources

- [ArchWiki: General recommendations](https://wiki.archlinux.org/title/General_recommendations)
- [ArchWiki: XDG user directories](https://wiki.archlinux.org/title/XDG_user_directories)
- [ArchWiki: Fonts](https://wiki.archlinux.org/title/Fonts)
- [ArchWiki: Font configuration](https://wiki.archlinux.org/title/Font_configuration)
- [ArchWiki: Bash](https://wiki.archlinux.org/title/Bash)
- [ArchWiki: Command-line shell](https://wiki.archlinux.org/title/Command-line_shell)
- [ArchWiki: Clipboard](https://wiki.archlinux.org/title/Clipboard)
- [ArchWiki: Archiving and compression](https://wiki.archlinux.org/title/Archiving_and_compression)
- [ArchWiki: Default applications](https://wiki.archlinux.org/title/XDG_MIME_Applications)
- [Arch manual: xdg-user-dirs-update(1)](https://man.archlinux.org/man/xdg-user-dirs-update.1)
- [Arch packages: xdg-user-dirs](https://archlinux.org/packages/extra/x86_64/xdg-user-dirs/)
- [Arch packages: wl-clipboard](https://archlinux.org/packages/extra/x86_64/wl-clipboard/)
- [Arch packages: noto-fonts](https://archlinux.org/packages/extra/any/noto-fonts/)
- [Arch packages: noto-fonts-cjk](https://archlinux.org/packages/extra/any/noto-fonts-cjk/)
- [Arch packages: noto-fonts-emoji](https://archlinux.org/packages/extra/any/noto-fonts-emoji/)
- [Arch packages: ttf-liberation](https://archlinux.org/packages/extra/any/ttf-liberation/)
- [Arch packages: bash-completion](https://archlinux.org/packages/extra/any/bash-completion/)
- [Arch packages: 7zip](https://archlinux.org/packages/extra/x86_64/7zip/)

## Next step

Continue with chapter 09 to select, install, and verify the daily graphical
applications: browser, file manager, PDF and image viewers, text editor, media
tools, and the initial development utilities. That chapter will also assign
the corresponding MIME and URL defaults.
