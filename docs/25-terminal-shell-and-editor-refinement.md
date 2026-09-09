# 25 — Refine Bash and the terminal editors

## Goal

Deploy the validated RogueOS terminal workflow without changing the login
shell or introducing a shell framework, editor plugin manager, or root-owned
user configuration. This chapter adds four independent GNU Stow packages:

- Bash history, Readline, aliases, Git context, exit status, and prompt;
- Nano editing defaults plus Arch and project-owned KDL highlighting;
- Micro behavior, the RogueOS palette, and KDL and Kitty syntax definitions;
- a plugin-free Vim learning configuration with persistent undo and a native
  Wayland clipboard provider.

The four implementations were committed separately in `niri-dotfiles` and
validated together on the first target ThinkPad on 2026-09-08.

## Boundaries

This chapter does not:

- run `chsh` or replace Bash;
- install Starship, Oh My Bash, Zsh, Fish, or Nushell;
- add Nano, Micro, Vim, or shell plugins from unreviewed sources;
- set a system-wide `EDITOR` or `VISUAL` policy;
- execute graphical-session daemons from `.bashrc`;
- place history, undo contents, swap files, credentials, or generated editor
  state in Git;
- replace Kitty's terminal configuration or shell-integration hooks.

Micro remains the approachable everyday terminal editor. Nano remains a small,
predictable fallback, especially in documentation and recovery work. Vim is a
secondary learning and recovery tool whose modal workflow can be learned
without a plugin ecosystem.

## Validated package set

The first target used:

```text
bash 5.3.15-1
nano 9.2-1
nano-syntax-highlighting 2025.07.01.r0.g256995b-2
micro 2.0.15-3
vim 9.2.1011-1
wl-clipboard 1:2.3.0-1
```

Inspect the current repository metadata and read Arch news before installing.
Then use one complete transaction:

```bash
sudo pacman -Syu \
    bash \
    nano \
    nano-syntax-highlighting \
    micro \
    vim \
    git \
    wl-clipboard
```

These packages have distinct roles:

| Package | Role in this chapter |
| --- | --- |
| `bash` | Existing login and interactive shell; no shell migration |
| `git` | Supplies the packaged `git-prompt.sh` loaded when present |
| `nano` | Small non-modal editor and built-in syntax definitions |
| `nano-syntax-highlighting` | Additional reviewed syntax files from Arch Extra |
| `micro` | Canonical approachable terminal editor |
| `vim` | Plugin-free modal learning and recovery editor |
| `wl-clipboard` | `wl-copy` and `wl-paste` for editor clipboard integration |

Confirm package ownership rather than downloading syntax files manually:

```bash
pacman -Q \
    bash git nano nano-syntax-highlighting micro vim wl-clipboard
pacman -Ql nano-syntax-highlighting | sed -n '1,20p'
command -v bash nano micro vim wl-copy wl-paste
```

## Select the reviewed dotfiles state

The four existing feature commits are intentionally separate:

```text
4a8a344 feat(nano): add editor configuration and KDL highlighting
cc82930 feat(micro): add editor configuration and custom syntax highlighting
4ba69b0 feat(bash): add shell configuration and RogueOS prompt
02820cf feat(vim): add terminal editor configuration
```

After the final documentation commit, create the cumulative
`post-install-25-v1` tag in both the post-install and dotfiles repositories.
For active development, first inspect the current checkout:

```bash
cd "$HOME/Projects/CycloniteRDX/niri-dotfiles"
git status --short --branch
git log --oneline --decorate -8
```

The working tree must be clean before deployment. A reconstruction following
this chapter should fetch tags and select the immutable chapter checkpoint
rather than assuming that a future `main` still represents this exact state.

## Preview and deploy the four Stow packages

Existing `~/.bashrc`, `~/.bash_profile`, or editor configurations may contain
unrecorded behavior. Inspect and back them up before Stow is allowed to create
links. Do not use `--adopt`: it can replace tracked repository contents with
the target's unreviewed files.

Preview all links:

```bash
cd "$HOME/Projects/CycloniteRDX/niri-dotfiles"

stow --simulate --verbose --no-folding --target="$HOME" \
    bash nano micro vim
```

Resolve every conflict deliberately. Then deploy:

```bash
stow --verbose --no-folding --target="$HOME" \
    bash nano micro vim
```

Verify exact link ownership:

```bash
for path in \
    "$HOME/.bash_profile" \
    "$HOME/.bashrc" \
    "$HOME/.config/nano/nanorc" \
    "$HOME/.config/nano/kdl.nanorc" \
    "$HOME/.config/micro/settings.json" \
    "$HOME/.config/micro/colorschemes/rogueos.micro" \
    "$HOME/.config/micro/syntax/kdl.yaml" \
    "$HOME/.config/micro/syntax/kitty.yaml" \
    "$HOME/.config/vim/vimrc"
do
    printf '%s -> %s\n' "$path" "$(readlink -f "$path")"
done
```

Every resolved target must remain below the reviewed `niri-dotfiles` clone.

## Bash policy

`.bash_profile` has one narrow responsibility: source `.bashrc` when it exists.
The interactive guard at the top of `.bashrc` prevents the remainder from
changing non-interactive scripts.

The configuration adds:

- append-only interactive history with consecutive duplicate suppression;
- bounded in-memory and on-disk history sizes;
- terminal-size correction after commands;
- prefix-aware Up and Down history search;
- case-insensitive and ambiguity-friendly completion;
- small color and directory-order aliases for `ls`, `ll`, `la`, and `grep`;
- Git branch and worktree indicators from Arch's packaged `git-prompt.sh`;
- the last nonzero command status;
- the compact two-line RogueOS prompt.

The status hook prepends its function to the existing `PROMPT_COMMAND` instead
of discarding other entries. This preserves Kitty's shell-integration hooks.
The configuration contains no command that starts a daemon or alters the Niri
session.

Validate syntax without sourcing either file into the current shell:

```bash
bash --noprofile --norc -n "$HOME/.bash_profile"
bash --noprofile --norc -n "$HOME/.bashrc"
```

Open a new Kitty window rather than blindly sourcing `.bashrc` over a possibly
different current state. Confirm the username, host, path, Git branch, dirty
state, and exit indicator. A bounded test is:

```bash
true
false
```

After `false`, the next prompt must show `✗ 1`; after the following successful
command, the indicator must disappear.

## Nano policy

The Nano package keeps editing non-modal and predictable. It enables automatic
indentation, four-space tabs, visual wrapping, line numbers, position and
scroll indicators, cursor/history persistence, and mouse interaction.

Syntax definitions are loaded in this order:

1. Nano's packaged definitions;
2. Nano's packaged `extra` definitions;
3. the Arch `nano-syntax-highlighting` collection;
4. the project-owned `~/.config/nano/kdl.nanorc` definition.

The final rule teaches Nano the `.kdl` extension and highlights comments,
strings, raw strings, numbers, booleans, properties, punctuation, and common
Niri sections. It does not modify the KDL document.

Open a real tracked file and confirm that the configuration loads without an
unknown-option or duplicate-syntax warning:

```bash
nano "$HOME/.config/niri/config.kdl"
```

## Micro policy

Micro uses its documented JSON configuration and project-owned syntax and
palette files. The selected policy enables external clipboard integration,
true color, search highlighting, diff gutter, trailing-whitespace and tab
warnings, persistent cursor and undo state, a scrollbar, visual word wrapping,
and four-space indentation.

Validate the JSON before opening the editor:

```bash
jq empty "$HOME/.config/micro/settings.json"
```

Then verify both custom syntax routes:

```bash
micro "$HOME/.config/niri/config.kdl"
micro "$HOME/.config/kitty/kitty.conf"
```

The RogueOS colorscheme and the matching KDL or Kitty highlighting must appear.
Copy and paste must reach the Wayland clipboard through `wl-clipboard`.

## Vim policy

Vim remains plugin-free. The configuration enables syntax and filetype-aware
indentation, true color, absolute line numbers, visual wrapping, four-space
indentation, incremental smart-case search, convenient split directions, a
compact status line, and the RogueOS palette over the packaged `habamax`
colorscheme.

Persistent undo and swap files are kept out of project directories. Create
their state directories before relying on them:

```bash
mkdir -p \
    "$HOME/.local/state/vim/undo" \
    "$HOME/.local/state/vim/swap"
```

The tracked configuration defines a Wayland provider through Vim's clipboard
provider interface when `wl-copy` and `wl-paste` are available. It falls back
to native clipboard support when appropriate and does not start a clipboard
manager.

Validate a non-interactive load:

```bash
vim -Nu "$HOME/.config/vim/vimrc" -n -es -c 'qa!'
printf 'vimrc_exit=%s\n' "$?"
```

The result must be zero. Open the KDL file, edit and save disposable content,
exercise the `+` clipboard register, and confirm that no undo or swap file is
created beside the project file:

```bash
vim "$HOME/.config/niri/config.kdl"
```

Do not make a real change merely to test Vim. Quit without writing after the
visual and clipboard checks unless an intended configuration edit exists.

## Integrated verification

Run the safe format and link checks:

```bash
cd "$HOME/Projects/CycloniteRDX/niri-dotfiles"

git status --short --branch
git diff --check
bash --noprofile --norc -n bash/.bash_profile
bash --noprofile --norc -n bash/.bashrc
jq empty micro/.config/micro/settings.json
vim -Nu vim/.config/vim/vimrc -n -es -c 'qa!'
```

On the first target, all tracked links resolved to the clone, both Bash files
passed syntax checking, Micro's JSON parsed, Vim loaded with exit status zero,
and both Vim state directories existed. Nano, Micro, and Vim were then
validated visually against the real Niri KDL configuration. New Kitty prompts
showed the RogueOS host/path layout and Git branch.

## Rollback

Remove only the four package links:

```bash
cd "$HOME/Projects/CycloniteRDX/niri-dotfiles"

stow --delete --verbose --target="$HOME" bash nano micro vim
```

This does not delete generated editor state or restore any pre-Stow files.
Restore a deliberately preserved backup only after confirming the Stow links
are gone. Package removal is optional: the applications remain usable with
their defaults when these user links are absent.

## Completion checklist

- [ ] All seven required packages are installed from official repositories.
- [ ] The four Stow packages deploy without adopting or overwriting files.
- [ ] Every tracked target resolves into the reviewed clone.
- [ ] Bash syntax checks pass and a new Kitty displays the expected prompt.
- [ ] Git and nonzero-exit prompt states are visible without breaking Kitty
      shell integration.
- [ ] Nano loads its packaged and project KDL syntax without warnings.
- [ ] Micro loads valid JSON, the RogueOS palette, KDL syntax, and Kitty syntax.
- [ ] Vim loads with exit status zero and uses separate undo and swap state.
- [ ] Nano, Micro, and Vim edit KDL correctly in a real terminal.
- [ ] Micro and Vim exchange text with the Wayland clipboard.
- [ ] No history, undo data, swap file, plugin state, or credential is tracked.
- [ ] The complete result is validated on the first ThinkPad.
- [ ] `post-install-25-v1` is created at the final documentation commits in
      both `arch-linux-post-install` and `niri-dotfiles`.

## Sources

- [GNU Bash manual](https://www.gnu.org/software/bash/manual/)
- [GNU Nano manual](https://www.nano-editor.org/dist/latest/nano.html)
- [Arch package: nano-syntax-highlighting](https://archlinux.org/packages/extra/any/nano-syntax-highlighting/)
- [Micro documentation](https://github.com/zyedidia/micro/tree/master/runtime/help)
- [Vim help](https://vimhelp.org/)
- [`wl-clipboard`](https://github.com/bugaevc/wl-clipboard)
- [GNU Stow manual](https://www.gnu.org/software/stow/manual/stow.html)

## Next step

Complete the cross-application GTK and Qt consistency review, then decide
whether Plymouth's existing graphical boot needs a restrained RogueOS theme.
Keep the textual fallback UKI independent throughout that evaluation.
