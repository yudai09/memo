# docker swarmを試す

## 第四回

https://leanpub.com/the-devops-2-1-toolkit

`Problems When Scaling Stateful Instances`

この内容について学習していく。
ＤＢとかSwarmの上でどう管理していくんだよという疑問が解決できることを期待して。

Before we proceed, I must state that there is no silver bullet that makes stateful services scalable and
fault-tolerant. Throughout the book, I will go through a couple of examples that might, or might
not, apply to your use case.

銀の弾丸はないけど、いろんな例を出していくそうだ。

ファイルに永続化する場合は、そのファイルをどうやってクラスタ間で共有するかという話がある。
NFSに乗せたらそこがシングルポイントになる。

consulに情報をいれるのが一番きれいだ。
consulは今のところSwarmっぽく使えるように作られていないのが難点らしい。
サンプルもやや泥臭い。

中途半端だけどここで寝るので章のはじめから読み直す。
