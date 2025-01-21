# Typical build
[text](https://docs.oracle.com/en/operating-systems/olcne/2.0/ockforge/install.html)ここの手順を実施して、`ock-forge`はインストール済みとする。また、このレポジトリにある`build.tar.gz`をオペレーターノードの`/root/images`配下に`base_image`というディレクトリとして展開済みとする。

## 事前準備
アーカイブを作成する際にNBDデバイスが必要になるので、下記の手順で用意する。
```
sudo modprobe nbd
lsmod | grep nbd # モジュールが正しくロードされたかを確認
ls /dev/nbd* # デバイスファイルが作成されているかを確認
```
もしやり直し等でNBDデバイスを使用してしまっている場合は、下記の手順でNBDデバイスを切断し、成果物も削除しておく。
```
qemu-nbd --disconnect /dev/nbd0
rm -rf out/1.30
```

## イメージのビルド
`ock-forge`のディレクトリに移動。
```
cd /root/ock-forge
```
`ock-forge`を実行。
```
./ock-forge -d /dev/nbd0 -D out/1.30/boot.qcow2 -i container-registry.oracle.com/olcne/ock-ostree:1.30 -O ./out/1.30/archive.tar -C ./ock -c configs/config-1.30 -P -p file
```

## nginxを含むイメージに再ビルド
`ock-forge`でビルドしたイメージには、`ocne image create`でビルドしたイメージとは異なりnginxは含まれない。OSTreeアーカイブをホストするコンテナイメージを作成するには、nginxを含むイメージに再ビルドが必要。

ここでは再ビルドをオペレーターノードで行うため、成果物をオペレーターノードにコピー。
```
scp -r out/1.30 oscip081:~/images
```

ここからはオペレーターノードで操作する。コピーしてきた成果物を`ocne image upload`コマンドでプライベートレジストリにプッシュする。
```
ocne image upload --type ostree --file archive.tar.gz --destination docker://oscip085:5000/olcne/ock-ostree:1.30 --arch amd64
```
`/root/images/base_image/build`に移動し、下記のコマンドでnginxを含むイメージを再ビルド。
```
podman build --tls-verify=false --no-cache --isolation chroot -t ock-ostree:1.30-nginx --build-arg OSTREE_IMG=ostree-unverified-registry:oscip085:5000/olcne/ock-ostree --build-arg PODMAN_IMG=oscip085:5000/olcne/ock-ostree:1.30 --build-arg ARCH=amd64 --build-arg HTTP_PROXY=10.115.208.19:80 --build-arg HTTPS_PROXY=10.115.208.19:80 --build-arg NO_PROXY=oscip085 $(pwd)
```
これで、ローカルレジストリに`ock-ostree:1.30-nginx`というイメージが作成される。このイメージをプライベートレジストリにプッシュ。
```
podman tag localhost/ock-ostree:1.30-nginx oscip085:5000/olcne/ock-ostree:1.30-nginx
podman push oscip085:5000/olcne/ock-ostree:1.30-nginx
```

## OSTreeアーカイブをホストするコンテナを起動
ここからはプライベートレジストリサーバのホスト（oscip085）で操作する。
先ほど再ビルドしたイメージを下記のコマンドで起動する。
```
podman run -d -p 8080:80 -v /root/ostree-repo:/usr/share/nginx/html/ks oscip085:5000/olcne/ock-ostree:1.30-nginx
```