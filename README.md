plugged: PKGBUILDs for $HOME
============================

Overview
--------

This branch contains PKGBUILDs for Vim and tmux plugins that
deliberately modify the user’s `$HOME` directory. This
is known to be incompatible with [Arch Packaging
Standards](https://wiki.archlinux.org/index.php/Arch_Packaging_Standards).

The aim here is to provide the very best possible plugin management
experience for Vim and tmux users.

Vim plugins will be completely self-contained in `~/.vim/plugged`,
and loaded with [plug](https://github.com/junegunn/vim-plug).

Tmux plugins will be completely self-contained in `~/.tmux/plugins`
and loaded manually with the tmux `run-shell` command.

Pacman is better than anything at cleanly installing, uninstalling,
updating, tracking, querying and organizing software. Pacman can also
improve security by verifying checksums and/or PGP signatures before
installing software to your system. Pacman, however, can only do so much
to customize the behavior of Vim and Tmux.

On the Vim side, [plug](https://github.com/junegunn/vim-plug) enables
lazy loading of plugins based on filetype among others.

On the Tmux side, the plugins made available by
[@tmux-plugins](https://github.com/tmux-plugins) open up a great deal
of new functionality.

In general, I’d rather install Vim and tmux plugins locally on a
per-user basis than load everything into `/usr/share` with `root:root`
permissions. That doesn’t make me want to give up Pacman, though.


Use cases
---------

In general, I’m a big supporter of the [AUR](https://aur.archlinux.org).
It’s the closest thing there is to an Amazon.com of FOSS. Uploading
software PKGBUILDs to the AUR makes it easy to search for, rate and
comment on the software.

Vim plugins often depend on other plugins being installed. While
[plug](https://github.com/junegunn/vim-plug) may provide the means
to declare plugin dependencies, Pacman offers a much better means of
tracking dependencies.

Vim plugins sometimes have a complex build procedure. While plug does
provide the means to script build procedures on a per-plugin basis,
PKGBUILDs offer higher clarity and expressiveness, as well as the means
to declare required and optional external software dependencies which
are then tracked by Pacman.

#### youcompleteme

Technically, it’s possible to script YouCompleteMe installation from
`vimrc`. The upstream build process isn’t set in stone though, which
means you might have to RTFM at some point. Its dependencies could change
at any time, and who’s to say the dependencies can’t have installation
quirks? I’m much more comfortable letting Pacman handle this one,
but that doesn’t mean I want to install the plugin to `/usr/share`
with `root:root` permissions.

Plugged package maintainers also benefit from having the `:PlugStatus`
command in Vim. This command returns a list of plugins that have been
updated upstream, which makes it faster to detect upstream changes to
the build process. For users of many Vim plugins (100+), the time savings
are substantial.

#### dutyl

Dutyl provides Vim support for dlang. It depends on several external
binaries, which in turn depend on other external binaries. Removing
any of these requisite binaries would break Dutyl. If you used Pacman
to install Dutyl, however, Pacman would prevent you from uninstalling
software required by Dutyl.

Also of interest is this line:

    groups=('vim-plugins')

This line, when added to a Vim plugin’s PKGBUILD, enables you to
rapidly query your system for all installed Vim plugins by running
`pacman -Qg vim-plugins | awk '{print $2}'`

    salt-vim-git
    vim-accordion-git
    vim-after-syntax-perl-git
    vim-after-syntax-vim-git
    vim-armasm-git
    vim-auto-pairs
    vim-awk-support
    #...

If something changes about the plugin’s build system upstream - a
new dependency is added, a Makefile is added, etc - you don’t have to
modify your `~/.vimrc`. Just wait for a new PKGBUILD to be uploaded to
the AUR and update your Vim plugins the same as you update your other AUR
packages. It’s faster and easier this way for everyone including those
who want to share a single `vimrc` across platforms (Windows/Mac/Linux).

If you want to develop on other platforms with a shared `vimrc`, and
you want to let plug handle the build procedure on those platforms,
you would add a post-update hook to `vimrc` along with rules to detect
`win` and `darwin`.

```vim
function! BuildYCM(info)
  if has("darwin")
    " do something
  elseif has("win")
    " do something else
  endif
endfunction

Plug 'Valloric/YouCompleteMe', { 'do': function('BuildYCM') }
```

#### fzf

[fzf](https://github.com/junegunn/fzf) is a fuzzy finder cmdline
utility that includes Vim support. The `fzf` command deserves to be
installed system-wide, but I’d rather `fzf.vim` be self-contained in
`~/.vim/plugged/fzf`.

#### pacstrapit

[pacstrapit](https://github.com/atweiden/pacstrapit) is an Arch system
setup script that enables fine-grained control over which packages are
installed. Users can opt for C/C++/Obj-C support, Go support, Python
support, Ruby support, Haskell support, etc. Opting for Haskell support
means installing Haskell-related Vim plugins. Opting out of Ruby support
means not installing Ruby-related Vim plugins. By setting environment
variables from `pacstrapit`, your Vim config can be dynamically altered
to support only what you need:

```vim

" Plugins {{{

silent! if plug#begin('~/.vim/plugged')
  if $haskell_support == 1
    " now plug won't attempt to manage hasksyn if you didn't opt for
    " Haskell support. The Haskell stuff won't touch your system.
    Plug 'travitch/hasksyn',  { 'for': 'haskell' }
  endif
call plug#end()
endif

" }}}

" Filetypes {{{

if $haskell_support == 1
  autocmd FileType haskell setlocal omnifunc=necoghc#omnifunc
  autocmd BufEnter *.hs call LoadHscope()
  set tags=tags;/,codex.tags;/
  set csprg=hscope
  set csto=1 " search codex tags first
  set cst
  set csverb
  set completeopt+=longest
endif

" }}}
```


Gotchas
-------

This strategy tricks [plug](https://github.com/junegunn/vim-plug) into
thinking that it is managing the lifecycle of all Vim plugins. Done
right, plug will maintain both the ability to lazy load plugins, and
the ability to query each plugin’s upstream git repository for updates
in parallel. Pacman will handle the actual installation, uninstallation
and updates.

To keep the system working, all plugin PKGBUILDs must be sourced from
git repos. The exact upstream name of the git repos must be preserved
in the resulting tarballs. That means introducing and using `$_gitname`
liberally in PKGBUILDs. It means no using `source=(${pkgname%-git}::...`
or `cd ${pkgname%-git}`. The `.git` directories must be kept in tact
inside `$pkgdir` so plug can query and check the status of upstream.

PKGBUILDs must `chown` all files in the `$pkgdir` to be
owned by the user running `makepkg`:

    find "$pkgdir" -exec chown -R $USER:users '{}' \;

`*.install` files will not be able to find where plugin documentation
is installed without knowing the user’s `$HOME` directory. While it
would be possible for Pacman to update plugin docs upon installation,
it would not be possible for Pacman to re-update plugin docs upon
uninstallation. Installation doc updates could work like this:

    echo -n "Updating vim help tags..."
    /usr/bin/vim --noplugins -u NONE -U NONE \
      --cmd ":helptags $_homedir/.vim/plugged/$_gitname/doc" --cmd ":q" > /dev/null 2>&1
    echo "done."

(Note: pacman -U must be run as root, so `$_homedir` needs to be
hardcoded in `*.install` during `makepkg`, likely with `sed`)

Upon uninstallation, `$_homedir/.vim/plugged/$_gitname/doc` would
no longer exist, and so Pacman would have no way to re-update Vim
helpdocs. It is better to run `:PlugUpdate` inside Vim. Plug will
automatically update your Vim helpdocs. All `vimdoc.install` files should
be removed or renamed accordingly.

All plugin files must be self-contained inside `~/.vim/plugged`. In
general:

    install -dm 755 "$pkgdir/$HOME/.vim/plugged/$_gitname"

[plug](https://github.com/junegunn/vim-plug) expects all plugins to
return their git repo origin URL in a specific form:

    x vim-wordy:
        Invalid URI: /home/user/.src/vim-wordy-USER-git/vim-wordy
        Expected:    https://git::@github.com/reedes/vim-wordy.git
        PlugClean required.

To work around this limitation:

    cd $_gitname
    git config remote.origin.url https://git::@github.com/reedes/vim-wordy.git


Other gotchas
-------------

The PKGBUILD’s `$pkgname` variable must include a unique string so
that multi-user systems can utilize the same plugged packages across
multiple user accounts without conflicts.

The obvious choice of unique string - including `$USER` in the package’s
name - has adverse privacy implications. I’ve instead chosen to take
the first eight digits of `$USER` double sha512’d.

    echo $USER | sha512sum | cut -c -128 | sha512sum | cut -c -8

Remember to update `~/.vimrc` to reflect the exact upstream name of the
git repos.

Also remember that plugged PKGBUILDs should, if anything, depend on
other plugged PKGBUILDS and not system-wide versions of plugins.


Example
-------

vim-wordy-plugged-git/PKGBUILD

```bash
pkgname=vim-wordy-plugged-$(echo $USER | sha512sum | cut -c -128 | sha512sum | cut -c -8)-git
_gitname=vim-wordy
pkgver=20140922
pkgrel=1
pkgdesc="Uncover usage problems in your writing ($USER)"
arch=('any')
depends=('vim')
makedepends=('git')
groups=('vim-plugins')
url="https://github.com/reedes/vim-wordy"
license=('MIT')
source=(git+https://github.com/reedes/$_gitname)
sha256sums=('SKIP')
install=wordy.install

pkgver() {
  cd $_gitname
  git log -1 --format="%cd" --date=short | sed "s|-||g"
}

prepare() {
  cd $_gitname
  git config remote.origin.url https://git::@${source##git+https://}.git
}

package() {
  cd $_gitname
  install -dm 755 "$pkgdir/$HOME/.vim/plugged/$_gitname"
  for _file in `ls -A`; do
    cp -dpr --no-preserve=ownership $_file "$pkgdir/$HOME/.vim/plugged/$_gitname"
  done
  find "$pkgdir" -mindepth 1 -maxdepth 1 -exec chown -R $USER:users '{}' \;
}
```

wordy.install

```bash
post_install() {
  printf "$wordy\n"
}

post_upgrade() {
  post_install
}

read -d '' wordy <<'EOF'
Wordy
=====
EOF
```
