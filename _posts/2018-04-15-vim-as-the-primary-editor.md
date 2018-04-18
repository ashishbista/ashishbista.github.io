---
layout: post
title: "Vim as the Primary Editor"
date: 2018-04-15 20:44:00 -0500
comments: true
categories: [vim, editor, ruby, tests, rspec]
---

Textmate used to be my favorite editor. I was an occasional Vim user at the same time. I switched to Sublime Text and used it for a small period because of its plugin support.

 I wanted to be productive as much as possible. I had that feeling while working with Vim. So, I decided to use Vim as my only editor. I uninstalled all other GUI-based editors. The year was 2013.

Nowadays, I use vim in my day-to-day work. I haven't tried Atom and Code yet. Indeed, I don't need them.

Now, I'm going to describe my journey with Vim.

Initially, I ended up maintaining a [.vimrc](https://github.com/ashishbista/vimi/blob/master/vimrc) file. I had key mappings and other configurations in it. I was manually installing the plugins I need. I felt like I was changing the .vimrc file so frequently. So, I was looking for a robust and ready-to-use Vim solution. I came across [Janus](https://github.com/carlhuda/janus). It had all the plugins I had installed on my own and a way to add my configurations.

I'm using Janus since I discovered it. I keep it up-to-date. As recommended  by Janus, I don't update the .vimrc file, rather I have .vimrc.before and [.vimrc.after](https://gist.github.com/ashishbista/a45f80c9161b3e8cb13223056bc6c670) files with my configurations. Their contents are shown below:

```
#.vimrc.before
let mapleader = ","
```

```
#.vimrc.after
set clipboard=unnamed  "Use system clipboard
set mouse=a                      " Use mouse
set autoread
set autochdir
color molokai
let g:vim_json_syntax_conceal = 0

" set cursorcolumn

set guifont=Inconsolata\ for\ Powerline:h15
let g:Powerline_symbols = 'fancy'
set encoding=utf-8
set t_Co=256
set fillchars+=stl:\ ,stlnc:\
set term=xterm-256color
set termencoding=utf-8

if has("gui_running")
   let s:uname = system("uname")
   if s:uname == "Darwin\n"
      set guifont=Inconsolata\ for\ Powerline:h13
   endif
endif

au BufNewFile,BufRead Jenkinsfile setf groovy
setlocal spell spelllang=en_us

set nocompatible              " be iMproved, required
filetype off                  " required

" set the runtime path to include Vundle and initialize
" set rtp+=~/.vim/bundle/Vundle.vim
" call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
 "Plugin 'VundleVim/Vundle.vim'

 " autocmd BufEnter * silent! lcd %:p:h

let NERDTreeIgnore += ['\.png$','\.jpg$','\.gif$','\.mp3$','\.flac$', '\.ogg$', '\.mp4$','\.avi$','.webm$','.mkv$','\.pdf$', '\.zip$', '\.tar.gz$', '\.rar$', '\.log$', '\.vagrant$', '\.docker$', 'node_modules', '_build', '*deps*']
set wildignore+=*.docker/*,*.vagrant/*,*/_build/*,*/deps/*,*/tmp/*,*.so,*.swp,*.zip     " MacOSX/Linux

let g:ctrlp_custom_ignore = {
  \ 'dir': '\.git$\|\.hg$\|\.svn|\.vagrant|\.docker$\|bower_components$\|dist$\|node_modules$\|project_files$\|_build$',
  \ 'file': '\v\.(exe|so|dll|log)$',
  \ 'link': 'bad_lin',
  \ }


" let g:airline_solarized_bg='dark'

let g:airline_powerline_fonts = 1

filetype plugin indent on

let g:airline_left_sep = ''
let g:airline_left_alt_sep = ''
let g:airline_right_sep = ''
let g:airline_right_alt_sep = ''
let g:airline_symbols.branch = ''
let g:airline_symbols.readonly = ''
let g:airline_symbols.linenr = '☰'
let g:airline_symbols.maxlinenr = ''

set guioptions=''

map <D-S-]> gt
map <D-S-[> gT
map <D-1> 1gt
map <D-2> 2gt
map <D-3> 3gt
map <D-4> 4gt
map <D-5> 5gt
map <D-6> 6gt
map <D-7> 7gt
map <D-8> 8gt
map <D-9> 9gt
map <D-0> :tablast<CR>

autocmd BufWinEnter * NERDTreeMirror
"autocmd VimEnter * NERDTree

" Swap files: https://coderwall.com/p/sdhfug/vim-swap-backup-and-undo-files
set undodir=~/.vim/.undo//
set backupdir=~/.vim/.backup//
set directory=~/.vim/.swp//
```

I have additional Vim plugins cloned inside `~/.janus` directory. Janus automatically loads the Plugins from this directory.

Vim has everything that I wish in an editor. It even allows me to switch between the source file and the test file very easily. With the help of [Vroom](https://github.com/skalnik/vim-vroom), I run tests from it. This increases the productivity because of little context switching. The best part is I don't miss my favorite editor while I'm remotely logged in to a server through SSH.

Indeed, I keep learning Vim every day.







