---
title: Vim Search Like a Boss
tags: vim search ctrpl find ag silver searcher fzf
---

## Vim search like a boss

[ctrlp.vim](https://github.com/kien/ctrlp.vim) war ein treuer Begleiter, doch
heute musste es weichen; [fzf](https://github.com/junegunn/fzf) ist - sofern
richtig konfiguriert - einfach schneller. Nicht, dass sich fzf bereits als
*command-line fuzzy finder* durchgesetzt hat, es lässt sich auch sehr einfach in
vim integrieren. 

Im Hintergrund verwendet fzf einfach nur
[`find`](https://www.gnu.org/software/findutils/). Der kleine Trick liegt nun
darin dies gegen [`ag`](https://github.com/ggreer/the_silver_searcher) zu
tauschen, welches um einiges schneller ist.

`ag` *(The Silver Searcher)* sollte im Paketmanager eurer Wahl zu finden sein. Die
Installation von fzf in diesem Fall über
[Vundle](https://github.com/VundleVim/Vundle.vim):

{% highlight vim %}
Plugin 'junegunn/fzf', { 'dir': '~/.fzf', 'do': './install --all' }
{% endhighlight %}

Zufälligerweise ist gerade eine geeignete Tastenkombination frei geworden. `ctrl + p`.

{% highlight vim %}
nnoremap <C-p> :FZF<CR>
{% endhighlight %}

fzf hört unter anderem auf die Umgebungsvariablen `FZF_DEFAULT_COMMAND` und
`FZF_CTRL_T_COMMAND`. Diese können wie folgt gesetzt werden:

{% highlight Shell linenos %}
export FZF_DEFAULT_COMMAND='ag --hidden --ignore .git -f -g ""'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"

# Es bietet sich an weitere Pfade zu ignorieren. 
# --ignore .cache --ignore .ccache --ignore .VirtualBox --ignore Steam
{% endhighlight %}

`--hidden` sorgt für die Beachtung von dotfiles. `-f` folgt Symlinks.  
So weit, so gut!  
Kommen wir nun zum besten Teil.
[ack.vim](https://github.com/mileszs/ack.vim) befähigt uns einer rekursiven
dateiübergreifenden musterbasierten Suche. Auch hier verwenden wir den *Silver
Searcher*.

{% highlight vim linenos %}
" Beides ist der .vimrc hinzuzufügen.

Plugin 'mileszs/ack.vim'

if executable('ag')
  let g:ackprg = 'ag --vimgrep'
endif
{% endhighlight %}

Öffnet vim in einem Projektordner eurer Wahl und probiert es aus.

{% highlight vim %}
:Ack! {pattern}
{% endhighlight %}

Es lohnt ein Blick in die
[README.md](https://github.com/mileszs/ack.vim/blob/master/README.md);
Insbesondere auf die [Keyboard
Shortcuts](https://github.com/mileszs/ack.vim/blob/master/README.md#keyboard-shortcuts).
