# ちょっとまえにやったsshのホスト名補完をもう少し高度にする

前回のやつ
http://kyudy.hatenablog.com/entry/2017/01/30/151721

参考にした記事
https://rcmdnk.com/blog/2015/05/14/computer-linux-mac-bash/

bash_completionを入れておくとknown_hostsと.ssh/configなどからホスト名を補完するようになって非常に便利。

## bash_completionをインストール
bashのバージョンが3.2だったのでbash_completionのバージョンは1.xとした。
```
wget https://github.com/scop/bash-completion/archive/1.x.zip  -O bash_completion.zip
unzip bash_completion.zip
mv bash-completion-1.x bash_completion
cd bash_completion
mkdir ~/bash_completion.d/
./install-completions ~/bash_completion/completions/ ~/bash_completion.d/
diff bash_completion bash_completion.bk
42,44c42,44
< [ -n "$BASH_COMPLETION" ] || BASH_COMPLETION=~/bash_completion/bash_completion
< [ -n "$BASH_COMPLETION_DIR" ] || BASH_COMPLETION_DIR=~/bash_completion.d
< [ -n "$BASH_COMPLETION_COMPAT_DIR" ] || BASH_COMPLETION_COMPAT_DIR=~/bash_completion.d
---
> [ -n "$BASH_COMPLETION" ] || BASH_COMPLETION=/etc/bash_completion
> [ -n "$BASH_COMPLETION_DIR" ] || BASH_COMPLETION_DIR=/etc/bash_completion.d
> [ -n "$BASH_COMPLETION_COMPAT_DIR" ] || BASH_COMPLETION_COMPAT_DIR=/etc/bash_completion.d

vi ~/.bashrc
# bash_completion
source bash_completion/bash_completion
```
