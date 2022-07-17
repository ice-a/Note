## 环境设置，vim中不能复制就是因为环境没设置好

.vimrc文件

```shell
syntax enable
syntax on
set number
set tabstop=4
set autoindent
set history=400
set smartindent


"配色
" Avoid clearing hilight definition in plugins
if !exists("g:vimrc_loaded")
   "Enable syntax hl
   syntax enable

   " color scheme
   if has("gui_running")
      set guioptions-=T "隐藏工具栏
      " set guioptions-=m
      " set guioptions-=L
      " set guioptions-=r
      "colorscheme  morning
      colorscheme zellner
      set guifont=YaHeiConsolasHybrid\ 10
      "hi normal guibg=#294d4a
   else
      "colorscheme  industry
      colorscheme zellner
   endif " has
endif " exists(...)
"set background=dark

"显示行号
"set nummber
set nu
set paste
set ignorecase

"tab替换为4空格
set ts=4
set expandtab

"语法高亮度显示
syntax enable
syntax on

"检测文件的类型 开启codesnip
filetype on
filetype plugin on
filetype indent on

match Todo /SW_64\|luoqiaoling\|mips64\|sw_64/

set showmatch
set smartindent
set shiftwidth=8
set tabstop=8
set autoindent
set ai!
set encoding=utf-8
set guifont=monospace\ 14
"set mouse=a
set linebreak
set hlsearch
set incsearch
set backspace=2

set helplang=cn

"在编辑过程中，在右下角显示光标位置的状态行
set ruler
set nolinebreak             " 在单词中间断行
" 在状态栏显示目前所执行的指令，未完成的指令片段亦
" 会显示出来
set showcmd                 
set wrap                    " 自动换行显示
"CmdLine settings {{{
set cmdheight=1             " 设定命令行的行数为 1
set laststatus=2            " 显示状态栏 (默认值为 1, 无法显示状态栏)
set statusline=%F%m%r,%Y,%{&fileformat}\ \ \ ASCII=\%b,HEX=\%B\ \ \ %l,%c%V\ %p%%\ \ \ [\ %L\ lines\ in\ all\ ]
                            " 设置在状态行显示的信息如下：
                            " %F    当前文件名
                            " %m    当前文件修改状态
                            " %r    当前文件是否只读
                            " %Y    当前文件类型
                            " %{&fileformat}
                            "       当前文件编码
                            " %b    当前光标处字符的 ASCII 码值
                            " %B    当前光标处字符的十六进制值
                            " %l    当前光标行号
                            " %c    当前光标列号
                            " %V    当前光标虚拟列号 (根据字符所占字节数计算)
                            " %p    当前行占总行数的百分比
                            " %%    百分号
                            " %L    当前文件总行数
"CmdLine settings End }}}

"显示匹配括号
set showmatch

"默认无备份
set nobackup
set nowritebackup

"在insert模式下能用删除键进行删除
set backspace=indent,eol,start

"去掉讨厌的有关vi一致性模式，避免以前版本的一些bug和局限
set nocp

" Line highlight 設此是游標整行會標註顏色
" set cursorline
" Column highlight 設此是遊標整列會標註顏色
"set cursorcolumn
"highlight CursorLine cterm=none ctermbg=2 ctermfg=0
let OmniCpp_DefaultNamespaces = ["std"]
let OmniCpp_GlobalScopeSearch = 1  " 0 or 1
let OmniCpp_NamespaceSearch = 1   " 0 ,  1 or 2
let OmniCpp_DisplayMode = 1
let OmniCpp_ShowScopeInAbbr = 0
let OmniCpp_ShowPrototypeInAbbr = 1
let OmniCpp_ShowAccess = 1
let OmniCpp_MayCompleteDot = 1
let OmniCpp_MayCompleteArrow = 1
let OmniCpp_MayCompleteScope = 1
set autoread                " 自动重新加载外部修改内容
"set paste

let Tlist_Show_One_File=1
let Tlist_Exit_OnlyWindow=1

let g:winManagerWindowLayout='FileExplorer|TagList'
nmap wm :WMToggle<cr>
let g:SuperTabRetainCompletionType=0
let g:miniBufExplMapCTabSwitchBufs=1
"let g:miniBufExplMapWindowNavArrows = 1
let g:miniBufExplMapWindowNavVim=1
filetype plugin indent on

nnoremap <silent> <F3> :Grep<CR>
nnoremap <silent> <F4> :Rgrep<CR>

imap <C-s> <Esc>:wa<cr>i<Right>
nmap <C-s> :wa<cr>
imap <C-q> <Esc>


"folding setting
"
nmap wv     <C-w>v
nmap wc     <C-w>c
nmap ws     <C-w>s


"quick fix
nmap <F5> :cn<cr> 
nmap <F6> :cp<cr>
"au VimLeave * mksession! ~/.vim/session/%:t.session
"au VimLeave * wviminfo! ~/.vim/session/%:t.viminfo
let g:vikiNameSuffix=".viki"
autocmd! BufRead,BufNewFile *.viki set filetype=viki

"ctags
"set tags=~/vmm/vmm/tags

"cscope
"if has("cscope")
"    set csprg=/usr/bin/cscope
"        set csto=0
"        set cst
"        set nocsverb
           " add any database in current directory
"        if filereadable("cscope.out")
"            cs add cscope.out
           " else add database pointed to by environment
"        elseif $CSCOPE_DB != ""
"            cs add $CSCOPE_DB
"        endif
"        set csverb
"    endif

"cs add ~/vmm/vmm/cscope.out ~/workarea/SRC/vmm/vmm

set cscopequickfix=s-,c-,d-,i-,t-,e-

nmap <C-_>s :cs find s <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>g :cs find g <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>c :cs find c <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>t :cs find t <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>e :cs find e <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>f :cs find f <C-R>=expand("<cfile>")<CR><CR>
nmap <C-_>i :cs find i ^<C-R>=expand("<cfile>")<CR>$<CR>
nmap <C-_>d :cs find d <C-R>=expand("<cword>")<CR><CR>

map <C-F12> :!ctags -R --c++-kinds=+p --fields=+iaS --extra=+q .<CR>

set completeopt-=preview
let g:vimwiki_list = [{'html_header': '~/vimwiki_html/header.tpl', 'html_footer': '~/vimwiki_html/footer.tpl'}]
nmap <C-e> :VimwikiAll2HTML<cr>:!wikipublish<cr>

"Restore cursor to file position in previous editing session
set viminfo='10,\"100,:20,n~/.viminfo
au BufReadPost * if line("'\"") > 0|if line("'\"") <= line("$")|exe("norm '\"")|else|exe "norm $"|endif|endif
```
