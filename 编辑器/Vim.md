## 环境设置

vim中选中右键没有复制是因为环境没设置好，解决方法一：shift+鼠标左键 方法二：set mouse=v

外部拷贝：方法一：选中，shift+Insert ，方法二：选中右键复制，方法三：选中ctrl+shift+c

取消行号`:set non :set nonumber`

搜索后`:noh :nohlsearch`取消高亮

vim光标修改。

ctags ：`ctags -R *`

`:qa` 同时退出所有

https://qa.1r1g.com/sf/ask/1959636591/

文本复制到vim格式错乱，使用`:set paste`再进入插入模式

粗光标永远看左侧

使用vi/vim打开文件是会生成.filename.swp文件用于存储缓冲区的内容（修改内容？），情况一：两人同时访问一个文件时会提示存在.filename.swp文件，建议使用可读浏览（enter键），直接编辑可能会出现版本冲突的情况（互相覆盖，会提示文件已经改动）。情况二：非正常退出，如果没有修改过，.filename.swp文件会自动删除，修改过，.filename.swp文件会保留，此时vim打开会提示存在.filename.swp文件，按e键删除（上一次修改不要了），按r键会根据.filename.swp文件恢复该文件，但不会自动删除.filename.swp，需要手动删除或者再打开一次按e键。

vimdiff相关：

```shell
# 跳转到下一个差异点
],c
# 跳转到上一个差异点
[,c
Ctrl-w, l：光标切换到右侧的窗口
Ctrl-w, h：光标切换到左侧的窗口
Ctrl-w, w：光标在两个窗口间彼此切换  
内容合并

可以使用 d, p （即 diff put）命令，将当前差异点中的内容覆盖到另一文件中的对应位置。
如当光标位于左侧文件（file1）中的第一行时，依次按下 d、p 键，则 file1 中的 Line one 被推送  

```

# 自用.vimrc

```vim
"match Todo /SW_64\|fy\|FIXME\|sw_64/
"显示行号
set nu 
"set nu=set number
"set nonu=set nonumber
"进入paste模式，vim不会启动自动缩进，而只是纯拷贝粘贴
set paste

"语法高亮度显示
"syntax enable 只在当前文件中有效
syntax on "对所有缓冲区中的文件有效

"高亮显示匹配括号
set showmatch

"默认无备份
set nobackup
set nowritebackup

"关闭vi兼容模式，避免以前版本的一些bug和局限
set nocp "set nocompatible

"开启相关插件
"侦测文件类型
filetype on
"载入文件类型插件
filetype plugin on
"为特定文件类型载入相关缩进文件
filetype indent on

"制表符tab显示宽度为4
set ts=4 "set tabstop=4
"用空格替换制表符tab
set expandtab

"为C程序提供自动缩进
set smartindent
set shiftwidth=4
"自动缩进，换行时，缩进量与上一行对齐
set autoindent

"激活鼠标的使用
set mouse=v
set selection=exclusive
set selectmode=mouse,key

"设置编码格式
set encoding=utf-8 " 设置 vim 展示文本时的编码格式
set fileencoding=utf-8    " 设置 vim 写入文件时的编码格式
"语言设置
set langmenu=zh_CN.UTF-8
"显示中文帮助
set helplang=cn

"设置字体 
set guifont=monospace\ 14

"自动换行显示
set wrap
"整词换行             
set linebreak

"搜索忽略大小写
set ignorecase
"高亮显示所有搜索到的内容，后面用map映射快捷键来方便关闭当前搜索的高亮
set hlsearch
"光标立刻跳转到搜索的内容
set incsearch
"搜索到最后匹配的位置后，再次搜索不回到第一个匹配处
set nowrapscan


"在insert模式下能用删除键进行删除
set backspace=indent,eol,start
set backspace=2

"历史记录数
set history=400

"共享剪贴板  
set clipboard=unnamed "+=？

"自动补全
inoremap ' ''<ESC>i
inoremap " ""<ESC>i
inoremap ( ()<ESC>i
inoremap [ [<ESC>i
inoremap < <><ESC>i
inoremap { {<CR>}<ESC>O
"设置跳出自动补全的括号
func SkipPair()
    if getline('.')[col('.') - 1] == '>' || getline('.')[col('.') - 1] == ')' || getline('.')[col('.') - 1] == ']' || getline('.')[col('.') - 1] == '"' || getline('.')[col('.') - 1] == "'" || getline('.')[col('.') - 1] == '}'
        return "\<ESC>la"
    else
        return "\t"
    endif
endfunc
"将tab键绑定为跳出括号
inoremap <TAB> <c-r>=SkipPair()<CR>

"打开文件类型检测, 加了这句才可以用智能补全
set completeopt=longest,menu

"在编辑过程中，在右下角显示光标位置的状态行
set ruler
set nolinebreak             "在单词中间断行
"在状态栏显示目前所执行的指令，未完成的指令片段亦会显示出来
set showcmd                 

"设置文件在外部被修改时，自动更新该文件
set autoread                

"恢复上次文件打开位置 
set viminfo='10,\"100,:20,n~/.viminfo
au BufReadPost * if line("'\"") > 0|if line("'\"") <= line("$")|exe("norm '\"")|else|exe "norm $"|endif|endif


""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"vim-plug设置
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
```

# .vimrc

```vim
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
set nu 
"set nu=set number
"set nonu=set nonumber
"进入paste模式，vim不会启动自动缩进，而只是纯拷贝粘贴
set paste

"语法高亮度显示
"syntax enable 只在当前文件中有效
syntax on "对所有缓冲区中的文件有效

"高亮显示匹配括号
set showmatch

"默认无备份
set nobackup
set nowritebackup

"关闭vi兼容模式，避免以前版本的一些bug和局限
set nocp "set nocompatible

"开启相关插件
"侦测文件类型
filetype on
"载入文件类型插件
filetype plugin on
"为特定文件类型载入相关缩进文件
filetype indent on

"match Todo /SW_64\|luoqiaoling\|mips64\|sw_64/

"制表符tab显示宽度为4
set ts=4 "set tabstop=4
"用空格替换制表符tab
set expandtab

"为C程序提供自动缩进
set smartindent
set shiftwidth=4
"自动缩进，换行时，缩进量与上一行对齐
set autoindent

"激活鼠标的使用
set mouse=a
set selection=exclusive
set selectmode=mouse,key

"设置编码格式
set encoding=utf-8 " 设置 vim 展示文本时的编码格式
set fileencoding=utf-8    " 设置 vim 写入文件时的编码格式
"语言设置
set langmenu=zh_CN.UTF-8
"显示中文帮助
set helplang=cn

"设置字体 
set guifont=monospace\ 14

"自动换行显示
set wrap
"整词换行             
set linebreak

"搜索忽略大小写
set ignorecase
"高亮显示所有搜索到的内容，后面用map映射快捷键来方便关闭当前搜索的高亮
set hlsearch
"光标立刻跳转到搜索的内容
set incsearch
"搜索到最后匹配的位置后，再次搜索不回到第一个匹配处
set nowrapscan


"在insert模式下能用删除键进行删除
set backspace=indent,eol,start
set backspace=2

"历史记录数
set history=400

"共享剪贴板  
set clipboard=unnamed +=？

"自动补全
inoremap ' ''<ESC>i
inoremap " ""<ESC>i
inoremap ( ()<ESC>i
inoremap [ [<ESC>i
inoremap < <><ESC>i
inoremap { {<CR>}<ESC>O
"设置跳出自动补全的括号
func SkipPair()
    if getline('.')[col('.') - 1] == '>' || getline('.')[col('.') - 1] == ')' || getline('.')[col('.') - 1] == ']' || getline('.')[col('.') - 1] == '"' || getline('.')[col('.') - 1] == "'" || getline('.')[col('.') - 1] == '}'
        return "\<ESC>la"
    else
        return "\t"
    endif
endfunc
"将tab键绑定为跳出括号
inoremap <TAB> <c-r>=SkipPair()<CR>

"打开文件类型检测, 加了这句才可以用智能补全
set completeopt=longest,menu

"在编辑过程中，在右下角显示光标位置的状态行
set ruler
set nolinebreak             "在单词中间断行
"在状态栏显示目前所执行的指令，未完成的指令片段亦会显示出来
set showcmd                 

"设置文件在外部被修改时，自动更新该文件
set autoread                

"恢复上次文件打开位置 
set viminfo='10,\"100,:20,n~/.viminfo
au BufReadPost * if line("'\"") > 0|if line("'\"") <= line("$")|exe("norm '\"")|else|exe "norm $"|endif|endif
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"命令行设置
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
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

"Line highlight 設此是游標整行會標註顏色
"set cursorline
"Column highlight 設此是遊標整列會標註顏色
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


let Tlist_Show_One_File=1
let Tlist_Exit_OnlyWindow=1

let g:winManagerWindowLayout='FileExplorer|TagList'
nmap wm :WMToggle<cr>
let g:SuperTabRetainCompletionType=0
let g:miniBufExplMapCTabSwitchBufs=1
"let g:miniBufExplMapWindowNavArrows = 1
let g:miniBufExplMapWindowNavVim=1


nnoremap <silent> <F3> :Grep<CR>
nnoremap <silent> <F4> :Rgrep<CR>

imap <C-s> <Esc>:wa<cr>i<Right>
nmap <C-s> :wa<cr>
imap <C-q> <Esc>


"折叠设置
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

"Generate tags and cscope.out from FileList.txt (c, cpp, h, hpp)
nmap <C-@> :!find -name "*.c" -o -name "*.cpp" -o -name "*.h" -o -name "*.hpp" > FileList.txt<CR>
                       \ :!ctags -L -< FileList.txt<CR>
                       \ :!cscope -bkq -i FileList.txt<CR>
                       \ :!rm FileList.txt<CR>

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"ctags
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
set tags=tags;"当前目录找不到就会去上层目录寻找
set autochdir

""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
"cscope
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
if has("cscope")
    set csprg=/usr/bin/cscope
        set csto=1
        set cst
        set nocsverb
        set cspc=3
           " add any database in current directory
        if filereadable("cscope.out")
            cs add cscope.out
           " else add database pointed to by environment
        elseif $CSCOPE_DB != ""
            cs add $CSCOPE_DB
        endif
        set csverb
endif
"加载cscope数据库
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

"map <C-F12> :!ctags -R --c++-kinds=+p --fields=+iaS --extra=+q .<CR>

set completeopt-=preview
let g:vimwiki_list = [{'html_header': '~/vimwiki_html/header.tpl', 'html_footer': '~/vimwiki_html/footer.tpl'}]
nmap <C-e> :VimwikiAll2HTML<cr>:!wikipublish<cr>
```
