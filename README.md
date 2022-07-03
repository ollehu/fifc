<div align="center">

# fifc 🐠

_fish fzf completions_

[![CI](https://github.com/gazorby/fifc/actions/workflows/ci.yml/badge.svg)](https://github.com/gazorby/fifc/actions/workflows/ci.yml)

</div>

fifc brings fzf powers on top of fish completion engine and allows customizable completion rules.

![gif usage](../assets/fifc.gif)

## Builtin features

### Preview/open

- Preview/open any file (text, image, gif, pdf, archive)
- Preview/open command's man page
- Preview/open full option description for commands (parse man pages)
- Preview/open directory content
- Preview/open process tree using
- Preview/open function definition

### Sources

- Use fd for path completion (recursively search for files and folders)

## 🚀 Install

```fish
fisher install gazorby/fifc
```

## 🔧 Usage

You only need to set one setting after install:

```fish
set -Ux fifc_editor <your-favorite-editor>
```

And enjoy built-in completions!

fifc can use modern tools if available:

| Prefer                                           | Fallback to | Used for                                  | Custom options     |
| ------------------------------------------------ | ----------- | ----------------------------------------- | ------------------ |
| [bat](https://github.com/sharkdp/bat)            | cat         | Preview files                             | `$fifc_bat_opts`   |
| [chafa](https://github.com/hpjansson/chafa)      | file        | Preview images, gif, pdf etc              | `$fifc_chafa_opts` |
| [hexyl](https://github.com/sharkdp/hexyl)        | file        | Preview binaries                          | `$fifc_hexyl_opts` |
| [fd](https://github.com/sharkdp/fd)              | find        | Complete paths                            | `$fifc_fd_opts`    |
| [exa](https://github.com/ogham/exa)              | ls          | Preview directories                       | `$fifc_exa_opts`   |
| [ripgrep](https://github.com/BurntSushi/ripgrep) | pcregrep    | Search options in man pages               | -                  |
| [procs](https://github.com/dalance/procs)        | ps          | Complete processes and preview their tree | `$fifc_procs_opts` |
| [broot](https://github.com/Canop/broot)          | -           | Explore directory trees                   | `$fifc_broot_opts` |

Custom options can be added for any of the commands used by fifc using the variable mentioned in the above table.

Example:

Show line number when previewing files:

- `set -U fifc_bat_opts --style=numbers`

Show hidden file by default:

- `set -U fifc_fd_opts --hidden`

## 🛠️ Write your own rules

Custom rules can easily be added using the `fifc` command. Actually, all builtin rules are added this way: see [conf.d/fifc.fish](https://github.com/gazorby/fifc/blob/52ff966511ea97ed7be79db469fe178784e22fd8/conf.d/fifc.fish)

See `fifc -h` for more details.

Basically, a rule allows you to trigger some commands based on specific conditions.

A condition can be either:

- A regex that must match commandline before the cursor position
- An arbitrary command that must exit with a non-zero status

If conditions are met, you can bind custom commands:

- **preview:** Command used for fzf preview
- **source:** Command that feeds fzf input
- **open:** Command binded to `fifc_open_keybinding` (defaults to ctrl-o)

All commands have access to the following variable describing the completion context:

| Variable           | Description                                                                           | Command availability |
| ------------------ | ------------------------------------------------------------------------------------- | -------------------- |
| `fifc_candidate`   | Currently selected item in fzf                                                        | all except source    |
| `fifc_commandline` | Commandline part before the cursor position                                           | all                  |
| `fifc_token`       | Last token from the commandline                                                       | all                  |
| `fifc_group`       | Group to which fish suggestions belong (can be either files, options or processes)    | all                  |
| `fifc_extracted`   | Extracted string from the currently selected item using the `extracted` regex, if any | all except source    |

### Matching order

fifc evaluate all rules in the order in which they have been defined and stops at the first where all conditions are met.
It does this each time it has to resolve source, preview and open commands.

Take the following scenario:

```fish
# Rule 1
fifc -n 'test "$fifc_group" = files' -p 'bat $fifc_candidate'
# Rule 2
fifc -n 'string match "*.json" "$fifc_candidate"' -p 'bat -l json $fifc_candidate'
```

When completing path, `$fifc_group` will be set to "files" so the first rule will always be valid in that case, and the second one will never be reached.

Another example:

```fish
# Rule 1
fifc --condition 'test "$fifc_group" = files' --preview 'bat $fifc_candidate'
# Rule 2
fifc --condition 'test "$fifc_group" = files' --source 'fd . --color=always --hidden $HOME'
```

Here, even if both rules have the same conditions, they won't interfere because fifc has to resolve source commands _before_ the preview commands, so order doesn't matter in this case.

### Override builtin rules

If you want to write your own rule based on the same conditions as one of the built-in ones, you can use fifc `--order` option.
It tells fifc to evaluate the rule in a predefined order, so you can set it to 1 to make sure it will be evaluated first.

When omitting the `--order`, the rule will be declared unordered, and will be evaluated _after_ all other ordered rules.

### Examples

Here is how the built-in rule for file preview/open is implemented:

```fish
fifc \
    # If selected item is a file
    -n 'test -f "$fifc_candidate"' \
    # bind `_fifc_preview_file` to preview command
    -p _fifc_preview_file \
    # and `_fifc_preview_file` when pressing ctrl-o
    -o _fifc_open_file
```

Interactively search packages in archlinux:

```fish
fifc \
    -r '^(pacman|paru)(\\h*-S)?\\h+\\w+' \
    -s 'pacman --color=always -Ss "$fifc_token" | string match -r \'^[^\\h+].*\'' \
    -e '.*/(.*?)\\h.*' \
    -f "--query ''" \
    -p 'pacman -Si "$fifc_extracted"'
```
