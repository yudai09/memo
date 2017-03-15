# Docker swarm modeのノード同士の通信をTLSで暗号化する

## 参照

https://docs.docker.com/v1.13/engine/swarm/

```
Secure by default: Each node in the swarm enforces TLS mutual authentication and encryption to secure communications between itself and all other nodes. You have the option to use self-signed root certificates or certificates from a custom root CA.
```

Swarmの通信を暗号化しようと思ってこの記事を書き始めたが、設定不要らしい。素晴らしい。
