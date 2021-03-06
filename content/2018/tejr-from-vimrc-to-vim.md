---
title: "From .vimrc to .vim"
publishDate: 2018-12-11
date: 2018-12-11
draft: true
description: "Breaking your long vimrc into a personal runtime directory."
slug: "from-vimrc-to-vim"
author:
  name: "Tom Ryder"
  email: "tom@sanctum.geek.nz"
---

From `.vimrc` to `.vim`
=======================

> You can’t have everything. Where would you put it?
>
> –Steven Wright

Attack of the 5,000-line vimrc
------------------------------

Vim is an endlessly configurable and extensible editor, with a culture of users
sharing configuration for their [`~/.vimrc` or `~/.vim/vimrc`][rc] startup
files. These files tend to expand over time. New users start by setting only a
few global defaults for options like [`'expandtab'`][et] and [`'wrap'`][wr],
and then add custom mappings, functions, filetype-specific logic, and
third-party plugins, often under an ever-shifting mantle of plugin managers.
Their vimrc files grow not only larger, but more intricate and complex.

If you have one of these longer files, you’re in good company. Damian Conway, a
Vim guru specializing in Perl, published [a vimrc with 1,855 lines][dc], and at
the time of writing, [Steve Losh’s is a whopping 3,160 lines][sl]. I’m sure you
can find vimrc files that are even longer—perhaps yours already is.

The issue with very long vimrc files isn’t the sheer amount of
configuration—after all, all of that power is there for a reason. However, if
you’ve been programming for a while, you’ll know from experience that it’s best
to avoid very large files with code that does many disparate things, because it
makes code hard to find, manage, and understand. Vim configuration is no
exception. Instead of a single large configuration file, there’s a case to be
made for having a set of smaller, well-organized files. Those files go in
`~/.vim`.

Creating a directory hierarchy in `~/.vim` to replace your large vimrc keeps
your configuration manageable. It improves efficiency by loading code only when
needed. It becomes clearer from a file’s position in the hierarchy what its
purpose is. It also makes it easier to *package* configuration for others to
use.

Your own personal `$VIMRUNTIME`
-------------------------------

Most of the benefit of an organized `~/.vim` directory comes from leaning on
Vim’s built-in behavior, which gives you a lot of control over how
configuration files are loaded.

Let’s start by looking at the structure of the runtime files that come with Vim
itself. You can find the path for this directory in the `$VIMRUNTIME` variable:

    :echo $VIMRUNTIME

If you’re using the version of Vim that was packaged with your operating
system, it will very likely be something like `/usr/share/vim/vim81`.

Let’s take a look at the contents of that directory:

    $ ls /usr/share/vim/vim81
    autoload/     bugreport.vim       colors/     compiler/
    defaults.vim  delmenu.vim         doc/        evim.vim
    filetype.vim  ftoff.vim           ftplugin/   ftplugin.vim
    ftplugof.vim  gvimrc_example.vim  indent/     indent.vim
    indoff.vim    keymap/             lang/       macros/
    menu.vim      mswin.vim           optwin.vim  pack/
    plugin/       print/              rgb.txt     scripts.vim
    spell/        synmenu.vim         syntax/     tools/
    tutor/        vimrc_example.vim

A quick-and-dirty count in the shell shows us there are 1,674 files in this
directory tree:

    $ find /usr/share/vim/vim81 -type f | wc -l
    1674

Of those, 1,335 are `.vim` files:

    $ find /usr/share/vim/vim81 -type f -name \*.vim | wc -l
    1335

All of these are just plain Vim script files, like your vimrc. Their location
within this directory determines when they are loaded. Only a few of them are
loaded on Vim startup. That’s well over a thousand files ready to be loaded
*only when relevant*. We should take a hint from Bram on that!

If we look at the value of the [`'runtimepath'`][ro] option in Vim, we can see
a few other paths:

    :set runtimepath?
      runtimepath=~/.vim,/usr/share/vim/vim81,…,~/.vim/after

The very first entry of `'runtimepath'` is `~/.vim`, and that’s where you can
build a structure mimicking that of `$VIMRUNTIME`. This is your *personal* Vim
runtime directory.

Lose the `:source`, Luke
------------------------

If you’ve worked with Vim script for a while, you probably know how to use the
[`:source`][sc] command to read it from a file. To load a separate file with
something like mapping definitions in it, you might have a line like this in
your vimrc:

    source ~/.vim/mappings.vim

Vim has another command named [`:runtime`][rt] for loading files that works
with the file layout of the `'runtimepath'` directories we just inspected. Used
without an exclamation mark, it reads Vim script commands from the first path
it finds in any of its `'runtimepath'` directories. With an exclamation mark
added, it reads *all* of them. In both cases, we can include filename pattern
matching with **globs**: `*` and `?` characters.

    runtime syntax/c.vim
    runtime! syntax/c.vim
    runtime! */maps.vim
    runtime! **/maps.vim

Note that we *don’t* include the leading `~/.vim` path in these patterns.

The [double asterisk][ss] in the last example here represents a set of
directory path elements, which can be up to 100 levels deep. This means that a
file in `~/.vim/foo/bar/baz/quux/maps.vim` would still be found and loaded.

Unlike `:source`, `:runtime` doesn’t raise errors if it can’t find any matching
files. This avoids boilerplate checks for file existence if a file’s absence is
not an error condition.

Much of Vim’s startup process is just thin wrappers around `:runtime` commands.
So are some of its other commands, including [`:filetype`][ft]. We can leverage
this to run our own code before, instead of, or after Vim’s bundled runtime
code, in order to disable, replace, modify, or extend it.

Turn on, `plugin`, drop out
---------------------------

You can start the process of breaking up your vimrc file by looking for blocks
of code that have expanded beyond simple configuration, and can be grouped
together meaningfully. We can extract these into self-contained files in the
`plugin` subdirectory.

As an example, if you read others’ vimrc files, you will often see approaches
to solving the problem of conveniently removing trailing whitespace at line
endings. Here’s one approach, in a function from the [Vim Tips wiki][vs]:

    function StripTrailingWhitespace()
      if !&binary && &filetype != 'diff'
        normal mz
        normal Hmy
        %s/\s\+$//e
        normal 'yz<CR>
        normal `z
      endif
    endfunction

This kind of function is usually followed by a mapping to call it:

    nnoremap <Leader>x :<C-U>call StripTrailingWhitespace()<CR>

This function doesn’t need to be loaded every time vimrc is sourced. Once
defined, it can just sit there, ready for calling when appropriate. In fact,
Vim throws an error if a vimrc with this function is reloaded. We could fix
that by declaring the function with `function!`, but there’s another way:
instead of putting the function definition in `~/.vimrc`, we can drop it into a
`.vim` file in `~/.vim/plugin`. We’ll use
`~/.vim/plugin/strip_trailing_whitespace.vim`.

Once this file is created, we can restart Vim, and then confirm our plugin has
been loaded by checking its path is in the output of [`:scriptnames`][sn]:

    :scriptnames
    ...
    10: ~/.vim/plugin/strip_trailing_whitespace.vim
    ...

Note that the `<Leader>x` mapping left in the vimrc still works, despite being
set *before* the function it calls was defined.

### What’s a plugin, anyway?

Why should we put the script in `~/.vim/plugin`? You might object that our
example isn’t a *real* plugin; it’s just a single function. However, that’s not
a meaningful distinction to Vim. At startup, it sources any and all `.vim`
files in the `plugins` subdirectory of each of the directories in
`'runtimepath'`. It makes no difference what those files actually contain. A
set of related abbreviations? Custom commands? Code dependent on one particular
machine or operating system? Sure, why not?

### Plugin subdirectories

Similarly, because `*.vim` files are loaded from the `plugin` directory
*recursively*, you can organize them in subdirectories if you want to:

    ~/.vim/plugin/insert/cancel.vim
    ~/.vim/plugin/insert/suspend.vim
    ~/.vim/plugin/visual/region.vim
    ~/.vim/plugin/whitespace/squeeze.vim
    ~/.vim/plugin/whitespace/trim.vim

The names of the subdirectories aren’t significant; Vim will search them all.
Remember how we mentioned Vim’s thinly-veiled `:runtime` wrappers? This is one
of them. A clue here is in the command that [`:help load-plugins`][lp] suggests
as an analogue to what Vim does internally at this step:

    :runtime! plugin/**/*.vim

### Local script scope

Putting blocks of code like this in distinct files in `~/.vim/plugin` has some
other advantages. One of these is Vim’s [script-variable][sv] **scoping** for
functions and variables that are only needed within the script:

    let s:myvar = 'value'
    function s:Myfunc()
      ...
    endfunction

This applies a unique prefix to all of your function names and variable names
at the time the file is sourced. That means you don’t have to worry about
trampling on any other variables defined elsewhere in your configuration.

There are some caveats here for defining mappings; make sure you read `:help
script-variable` carefully to make sure you understand how to use [`<SID>`
prefixes][si].

### Short-circuiting and load guards

Another advantage of separate script files is the ability to **short-circuit**
a script, to prevent it from loading if it’s not appropriate to do so. This is
done by checking at the start of the script whether the rest of it should be
loaded, and skipping it with [`:finish`][fn] if it shouldn’t.

We can use this to check options like [`'compatible'`][co], the Vim version
number, the availability of a feature, or whether the plugin has already been
loaded:

    if &compatible
          \ || v:version < 700
          \ || has('folding')
          \ || exists('g:loaded_myplugin')
      finish
    endif
    let g:loaded_myplugin = 1

This way, you don’t have to wrap all your feature-dependent code in clumsy
[`:if`][if] blocks.

### The question of mappings

Should you include a mapping that uses a defined function in the plugin itself?
It’s up to you, but the author likes to think of vimrc files as where
user-level preferences go, and plugins where the code they call should go.
Mapping choices are personal, and fall into the former category.

If you want to keep some abstraction between what the plugin does and how it’s
called, you can use [`<Plug>` prefix][pp] mappings to expose an interface from
the plugin file:

    function s:StripTrailingWhitespace()
      ...
    endfunction
    nnoremap <Plug>StripTrailingWhitespace
          \ :<C-U>call <SID>StripTrailingWhitespace()<CR>

You can then put your choice of mapping for that target in your vimrc:

    nmap <Leader>x <Plug>StripTrailingWhitespace

If someone else wants to use your plugin, this makes choosing their own
mappings for it more straightforward. There’s more general advice about good
mapping practices in writing fully-fledged plugin files in [`:help
write-plugin`][wp].

Not really my `:filetype`
-------------------------

Another pattern in big vimrc files is setting options only for buffers of a
certain filetype. For example, this line of code is intended to set the
[`'spell'`][sp] option, to highlight possible spelling errors in the text, but
*only* for [`mail` filetype][mf] buffers:

    autocmd FileType mail setlocal spell

The first thing to note here is that this should be surrounded in a
self-clearing [`augroup`][ag], so that reloading it doesn’t make multiple
definitions for the same hook:

    augroup ftmail
      autocmd!
      autocmd FileType mail setlocal spell
    augroup END

This is annoying, but there’s a way to avoid this boilerplate.

The second thing to note about our `autocmd` is that it’s set every time vimrc
is loaded, *regardless* of whether a mail file is actually edited in that
session. It therefore makes more sense to put this into a [filetype plugin][fp]
or **ftplugin**, so that it’s only loaded when relevant.

The `autocmd` hooks that set a buffer’s filetype are defined in
`$VIMRUNTIME/filetype.vim`. They apply heuristics to guess and then set the
type of a buffer, and then Vim runs any appropriate filetype plugins
afterwards: files in `'runtimepath'` directories named `ftplugin/FILETYPE.vim`
will be sourced.

This means there’s no need for the `autocmd` hooks around our `'spell'`
setting. We already have hooks for changes of filetype available to us, and we
can just put this single line in `~/.vim/ftplugin/mail.vim` to use them:

    setlocal spell

With this done, upon editing a new `mail` buffer, we can confirm that our
filetype plugin was loaded when the filetype was chosen using `:scriptnames`:

    :set filetype=mail
    :scriptnames
    ...
    20: ~/.vim/ftplugin/mail.vim
    ...
    :set spell?
      spell

This is better, but we can improve it further.

### Loading filetype configuration afterwards

Rather than putting our `'spell'` setting in `~/.vim/ftplugin/mail.vim`, we can
put it in `~/.vim/after/ftplugin/mail.vim`—note the extra directory named
`after` in the path.

Files in the [`after` runtime directory][ad] are loaded *after* the analogous
runtime files included in Vim. Using this path, we can ensure that our option
is set *after* the `mail` filetype plugin in `$VIMRUNTIME/ftplugin/mail.vim`
has been sourced. This is how to *override* something a filetype plugin does if
you don’t like it.

### Breaking up filetype plugins

If you need to make this even more granular, you can also put files in
*subdirectories* named after the filetype:

    ~/.vim/after/ftplugin/mail/spell.vim
    ~/.vim/after/ftplugin/mail/quote.vim

The filetype followed by an underscore and then a script name works, too:

    ~/.vim/after/ftplugin/mail_spell.vim
    ~/.vim/after/ftplugin/mail_quote.vim

You may have guessed by now that filetype switching is yet another `:runtime`
wrapper. Switching to a filetype of `mail` effectively runs this command:

    :runtime! ftplugin/mail.vim ftplugin/mail_*.vim ftplugin/mail/*.vim

### Undoing filetype settings

If the filetype of a buffer changes, we should *reverse* any local
configuration we applied. We can do this with the [`b:undo_ftplugin`][uf]
variable, which contains a list of pipe-separated (`|`) commands. When a
buffer’s filetype changes, the commands *undo* the buffer-specific settings for
the previous filetype, ready for the new filetype’s plugins to be loaded.

After each filetype plugin setting we make, we should append corresponding
commands to reverse that change to `b:undo_ftplugin`. For our `'spell'`
example, we’d do this:

    setlocal spell
    let b:undo_ftplugin .= '|setlocal spell<'

The `spell<` syntax used here, with a trailing left angle bracket, specifies
that the local value of `'spell'` should be restored to match the global value
of `'spell'` when the `mail` filetype is unloaded.

After setting a filetype, we can check the `b:undo_ftplugin` variable’s value
with [`:let`][lt]:

    :set filetype=mail
    :let b:undo_ftplugin
    b:undo_ftplugin        setl modeline< tw< fo< comments<|setlocal spell<

### The difference with indent

Filetype-specific code related to indentation goes in a different location
again: `~/.vim/indent/FILETYPE.vim` or `~/.vim/after/indent/FILETYPE.vim`.
Those files are sourced if you include the word `indent` in your `vimrc`’s
`:filetype` call. You should use this layout for files that change
[`'autoindent'`][ai] or [`'indentexpr'`][ie] settings, for example.

You can put indent settings in your filetype plugin if you want to, but
remember that we’re trying to find the *right place* for things. Doing it this
way keeps your indentation settings separate from all other filetype-specific
settings. This gives users an easy way to load only what they want, using
appropriate arguments to their vimrc’s `:filetype` call.

### Detecting filetypes

As a final note for filetype-dependent logic, if you have any hooks to *set* a
buffer’s filetype in the first place, based on its filename or contents, those
go in the [`ftdetect`][fd] directory. You might put this in
`~/.vim/ftdetect/irssilog`, for example:

    autocmd BufNewFile,BufRead */irc/*.log setfiletype irssilog

Putting the hooks in `~/.vim/ftdetect` means they are sourced as part of the
`filetypedetect` `augroup` defined in `filetype.vim`. This is another context
in which you don’t need to surround `autocmd` definitions in a self-clearing
`augroup`, because it’s already been done for you.

Be water, my friend
-------------------

All of the above is just the beginning. We haven’t even touched on
[lazy-loading functions][al] for speed with definitions in `~/.vim/autoload`,
or custom [`:compiler`][cm] definitions for setting [`'makeprg'`][mp] and
[`'errorformat'`][ef] in `~/.vim/compiler`. These are yet more examples of Vim
functionality that wraps around `:runtime` loading.

While Vim gives you a lot of flexibility in configuring and customizing, there
is definitely a **Way of Vim** for the timely loading of relevant
configuration, and if you learn a little about how it works, you’ll fight with
your editor that much less. If this seems stringent to you, think back to when
you first learned Vim. Do you remember how strange using the HJKL keys for
movement seemed, before it made sense? Do you remember how you wanted to stay
in insert mode all the time, before normal mode made sense?

Working within the Vim runtime file structure instead of ignoring it or
fighting with it makes your `~/.vim` directory into a refined toolbox, with a
place for everything, and everything in its place. It’s well worth the effort!

If you’d like to see an example of how this layout can end up looking when you
make it work for you, [the author’s personal `~/.vim` directory is available
for download][pv].

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

[ad]: https://vimhelp.appspot.com/options.txt.html#after-directory
[ag]: https://vimhelp.appspot.com/autocmd.txt.html#%3Aaugroup
[ai]: https://vimhelp.appspot.com/options.txt.html#%27autoindent%27
[al]: https://vimhelp.appspot.com/eval.txt.html#autoload
[cm]: https://vimhelp.appspot.com/quickfix.txt.html#%3Acompiler
[co]: https://vimhelp.appspot.com/options.txt.html#%27compatible%27
[dc]: https://github.com/thoughtstream/Damian-Conway-s-Vim-Setup/blob/cbe1fb5b5505e17bd7709669168c367903d94cd4/.vimrc
[ef]: https://vimhelp.appspot.com/options.txt.html#%27errorformat%27
[et]: https://vimhelp.appspot.com/options.txt.html#%27expandtab%27
[fd]: https://vimhelp.appspot.com/filetype.txt.html#ftdetect
[fn]: https://vimhelp.appspot.com/repeat.txt.html#%3Afinish
[fp]: https://vimhelp.appspot.com/usr_05.txt.html#ftplugins
[ft]: https://vimhelp.appspot.com/filetype.txt.html#%3Afiletype
[ie]: https://vimhelp.appspot.com/options.txt.html#%27indentexpr%27
[if]: https://vimhelp.appspot.com/eval.txt.html#%3Aif
[lp]: https://vimhelp.appspot.com/starting.txt.html#load-plugins
[lt]: https://vimhelp.appspot.com/eval.txt.html#%3Alet
[mf]: https://vimhelp.appspot.com/syntax.txt.html#mail%2Evim
[mp]: https://vimhelp.appspot.com/options.txt.html#%27makeprg%27
[pp]: https://vimhelp.appspot.com/usr_41.txt.html#using-%3CPlug%3E
[pv]: ../tejr-from-vimrc-to-vim/tejr-from-vimrc-to-vim.zip
[rc]: https://vimhelp.appspot.com/usr_05.txt.html#vimrc-intro
[ro]: https://vimhelp.appspot.com/options.txt.html#%27runtimepath%27
[rt]: https://vimhelp.appspot.com/repeat.txt.html#%3Aruntime
[sc]: https://vimhelp.appspot.com/repeat.txt.html#%3Asource
[si]: https://vimhelp.appspot.com/map.txt.html#%3CSID%3E
[sl]: https://bitbucket.org/sjl/dotfiles/src/e2a961f1d037e53ea2809885a65feba66a9aa03e/vim/vimrc?at=default&fileviewer=file-view-default
[sn]: https://vimhelp.appspot.com/repeat.txt.html#%3Ascriptnames
[sp]: https://vimhelp.appspot.com/options.txt.html#%27spell%27
[ss]: https://vimhelp.appspot.com/editing.txt.html#starstar-wildcard
[sv]: https://vimhelp.appspot.com/eval.txt.html#script-variable
[uf]: https://vimhelp.appspot.com/usr_41.txt.html#undo_ftplugin
[wp]: https://vimhelp.appspot.com/usr_41.txt.html#write-plugin
[wr]: https://vimhelp.appspot.com/options.txt.html#%27wrap%27
[vs]: http://vim.wikia.com/wiki/Remove_unwanted_spaces
