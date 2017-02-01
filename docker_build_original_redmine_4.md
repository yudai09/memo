# Redmineのバージョンアップに伴う添付ファイルのマイグレーション

現行の環境のfilesをS3に移植したい。
s3のプラグイン `ka8725/redmine_s3` にマイグレーション用のコマンドが用意されているので、それをつかう。
現行の古いRedmineにこのプラグインを入れるのはおそらく難しいので、新しく構築したサーバにファイルをおいて実行することにする。

容量が1.7GBもあるので、一旦圧縮してもっていくことにした。
```
zip -r /usr/local/redmine/files/files.zip /usr/local/redmine/files/
```

```
du -sh /usr/local/redmine/files/files.zip                                                                                     1.2G    /usr/local/redmine/files/files.zip
```
そんなに小さくならない・・・

## 検証開始

とりあえず以下のファイルを持っていってマイグレーションを検証するところから始める。
```
170105134250_f066e22937365a6eac1ccfc68e982807.log
```

コンテナの中にcopyで配置
```
# ls files/170105134250_f066e22937365a6eac1ccfc68e982807.log
files/170105134250_f066e22937365a6eac1ccfc68e982807.log
# rake redmine_s3:files_to_s3
/usr/local/bundle/gems/htmlentities-4.3.1/lib/htmlentities/mappings/expanded.rb:465: warning: duplicated key at line 466 ignored: "inodot"
Put file 170105134250_f066e22937365a6eac1ccfc68e982807.log
```

s3のバケットに挿入されたことが確認できた。
簡単。
