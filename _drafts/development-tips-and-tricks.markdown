---
layout: post
title: "Development Tips & Tricks"
date: 2020-01-17 19:56:00 -0800
categories: development
---
This is a collection of some cool tips and tricks related to software development and computer usage in general.

## Cool bash/zsh aliases and functions

### Move up a single directory

```bash
up() {
	cd $(eval printf '../'%.0s {1..$1})
}
```

### Make a directory and `cd` into it

```bash
mkcd() {
	if [ $# != 1 ]; then
		echo "Usage: mkcd <dir>"
	else
		mkdir -p $1 && cd $1
	fi
}
```

### Make a directory and take ownership with `sudo`

```bash
mkchown() {
	if [ $# != 1 ]; then
		echo "Usage: mkchown <dir>"
	else
		sudo mkdir -p $1 && sudo chown $(whoami):$(whoami) $1
	fi
}
```

### Delete local and remote git refs/branches at the same time

**Careful with this one!**

```bash
git-nuke() {
	git branch -D $1 && git push origin :$1
}
```

### Use `jq` with both JSON and non-JSON lines.

I often pipe command output to [`jq`](https://github.com/stedolan/jq), but it chokes on any line that isn't JSON. Use `jjq` instead to catch those errors and print them normally.

```bash
jjq() {
	jq -R -r "${1:-.} as \$line | try fromjson catch \$line"
}
```

### Alias over an existing command, only if an executable exists

This one is especially helpful if you use your bash/zsh configuration on multiple machines. 

```bash
[ "$(command -v bat)" ] && alias cat="bat"
```

<!--

# easy kill processes with fzf
kill_process() {
	local pid=$(ps -ef | sed 1d | eval "fzf ${FZF_DEFAULT_OPTS} -m --header='[kill:process]'" | awk '{print $2}')

	if [ "x$pid" != "x" ]
	then
		echo $pid | xargs kill -${1:-9}
		kp
	fi
}

# brew install with fzf
brew_install_fzf() {
	local inst=$(brew search | eval "fzf ${FZF_DEFAULT_OPTS} -m --header='[brew:install]'")

	if [[ $inst ]]; then
		for prog in $(echo $inst)
		do brew install $prog
		done
	fi
}

# checkout git branch (including remote branches), sorted by most recent commit, limit 30 last branches
fbr() {
	local branches branch
	branches=$(git for-each-ref --count=30 --sort=-committerdate refs/heads/ --format="%(refname:short)") &&
	branch=$(echo "$branches" |
			 fzf-tmux -d $(( 2 + $(wc -l <<< "$branches") )) +m) &&
	git checkout $(echo "$branch" | sed "s/.* //" | sed "s#remotes/[^/]*/##")
}

# autojump when used with no args uses fzf
j() {
	if [[ "$#" -ne 0 ]]; then
		cd $(autojump $@)
		return
	fi
	cd "$(autojump -s | gsed '/_____/Q; s/^[0-9,.:]*\s*//' | fzf --height 40% --reverse --inline-info)"
}

# git interactive rebase with fzf commit selection
girb() {
	git rebase -i $(git log --decorate --oneline --color=always | fzf --ansi | cut -d ' ' -f1 )^
}

# browse chrome history
c() {
	local cols sep google_history open
	cols=$(( COLUMNS / 3 ))
	sep='{::}'

	if [ "$(uname)" = "Darwin" ]; then
	google_history="$HOME/Library/Application Support/Google/Chrome/Default/History"
	open=open
	else
	google_history="$HOME/.config/google-chrome/Default/History"
	open=xdg-open
	fi
	cp -f "$google_history" /tmp/h
	sqlite3 -separator $sep /tmp/h \
	"select substr(title, 1, $cols), url
	 from urls order by last_visit_time desc" |
	awk -F $sep '{printf "%-'$cols's  \x1b[36m%s\x1b[m\n", $1, $2}' |
	fzf --ansi --multi | gsed 's#.*\(https*://\)#\1#' | xargs $open > /dev/null 2> /dev/null
}

# Select a docker container to start and attach to
function da() {
	local cid
	cid=$(docker ps -a | sed 1d | fzf -1 -q "$1" | awk '{print $1}')

	[ -n "$cid" ] && docker start "$cid" && docker attach "$cid"
}

# Select a running docker container to stop
function ds() {
	local cid
	cid=$(docker ps | sed 1d | fzf -q "$1" | awk '{print $1}')

	[ -n "$cid" ] && docker stop "$cid"
}

# find in zsh history
fh() {
	print -z $( ([ -n "$ZSH_NAME" ] && fc -l 1 || history) | fzf +s --tac | gsed -r 's/ *[0-9]*\*? *//' | gsed -r 's/\\/\\\\/g')
}

dsa() {
	docker stop $(docker ps -a -q)
}

bay-clone() {
	git clone git@bitbucket.org:bayphotolab/$1.git
}

unsetopt AUTOcd
setopt noflowcontrol

export GPG_TTY=$(tty)
if [[ -n "$SSH_CONNECTION" ]]; then
	export PINENTRY_USER_DATA="USE_CURSES=1"
fi

# Collides with "bayphoto" gem excecutable
unalias bp

[ "$(command -v nvim)" ] && export EDITOR="$(which nvim)"
[ "$(command -v bat)" ] && alias cat="bat"
[ "$(command -v hub)" ] && alias git="hub"
[ "$(command -v hub)" ] && alias gci="hub ci-status -v"
[ "$(command -v exa)" ] && alias ls="exa"
[ "$(command -v nvim)" ] && alias vim="nvim"
[ "$(command -v thefuck)" ] && eval $(thefuck --alias)

[ -f /Users/taylor/.travis/travis.sh ] && source /Users/taylor/.travis/travis.sh

alias kp="kill_process"
alias bip="brew_install_fzf"
alias dc="docker-compose"
alias gst="git status -s"
alias bid="bundle install --path=vendor --jobs=$(sysctl -n hw.ncpu) --binstubs=.bundle/bin"
alias gfap="git fetch --all --prune"
alias glog="thicket --color-prefixes --refs | less"
alias gloga="thicket --color-prefixes --all --refs | less"
alias wglog="watch --color -n 1 thicket --color-prefixes -n 200 --refs"
alias wgloga="watch --color -n 1 thicket --color-prefixes -n 200 --refs --all"
alias xit="exit"
alias work="nohup kitty --session ~/.dotfiles/work-kitty &"
alias rake="noglob rake"

if [ "$(command -v kitty)" ]; then
	alias aedirnlan="kitty +kitten ssh taylor@aedirn.local"
	alias aedirnwan="kitty +kitten ssh taylor@home.thurlow.io"
	alias cintralan="kitty +kitten ssh taylor@cintra.local"
	alias cintrawan="kitty +kitten ssh taylor@home.thurlow.io -t ssh taylor@cintra.local"
	alias whatbox="kitty +kitten ssh frizkie@apollo.whatbox.ca"
else
	alias aedirnlan="ssh taylor@aedirn.local"
	alias aedirnwan="ssh taylor@home.thurlow.io"
	alias cintralan="ssh taylor@cintra.local"
	alias cintrawan="ssh taylor@home.thurlow.io -t ssh taylor@cintra.local"
	alias feral="ssh frizkie@selene.feralhosting.com"
	alias whatbox="ssh frizkie@apollo.whatbox.ca"
fi

[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh

get_ruby() {
	$(which ruby) <<RUBY
		$:.unshift File.join(Dir.home, ".dotfiles", "ruby_scripts")
		ARGV = ["$2"]
		$1
RUBY
}

-->
