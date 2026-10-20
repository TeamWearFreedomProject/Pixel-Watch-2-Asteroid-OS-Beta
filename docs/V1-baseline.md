# V1の実物確認とAurora UIへの移行方針

2026-10-20（日本時間）に、[V1 Release](https://github.com/TeamWearFreedomProject/Pixel-Watch-2-Asteroid-OS-Beta/releases/tag/V1)の4資産を取得し、GitHubが返したSHA256と全ファイルを照合しました。起動イメージ・ramdisk・rootfsをオフラインで読み、Watchへの書き込みや実機起動は行っていません。V1の配布イメージも変更していません。

## 確認した構成

| 項目 | 実物から確認した内容 |
| --- | --- |
| OS | AsteroidOS 2.2-nightly、BUILD_ID 20261005064926 |
| カーネル | ARM64 Linux 5.15.144 |
| userspace | ARM32 hard-float、glibc（opkgのELFとローダーを確認） |
| UIとパッケージ | Qt 6、asteroid-launcher、opkgが存在 |
| boot.img | header v4、29,319,168 bytes、kernelのみ |
| init_boot.img | header v4、初期起動用LZ4 ramdisk |
| vendor_kernel_boot.img | vendor header v4、DTBと141個のカーネルモジュール |
| 主要モジュール | msm_drm.ko、touch_nt38350.ko、Google battery/charger、SMBlite |
| rootfs | ext4、1,441,935,360 bytes |

全資産のURL・サイズ・SHA256は[v1-assets.json](v1-assets.json)に記録しています。ReleaseのタグはREADMEのコミットを指していますが、そこから元のビルド用レイヤーやカーネルのソースコミットは特定できません。過去ログのレイヤー版を、このV1のビルド元と決めつけないでください。

## 起動条件として確認が必要な箇所

初期ramdiskのmachine.confはuserdataを `mmcblk0p82` と指定しています。`/init` はその領域へ `fsck.ext4 -p` を実行し、読み書き可能でマウントした後、`asteroidos.ext4` を探します。rootfsもfsckし、loopデバイスで読み書き可能にマウントします。単独のRAM診断イメージと同じ動作ではありません。

rootfsの `aurora-vendor-mount.sh` は `/dev/mmcblk0p80` 内のsystem_b/vendor_b/vendor_dlkm_b/system_dlkm_bを固定オフセットからdm-linearで構成し、Android領域を読み取り専用でマウントします。コードには、UBports実行時のslot B配置に基づくことと、super配置変更時には再取得が必要なことが記載されています。現在のWatchがslot Bであることだけでは、オフセットの一致は証明できません。

そのAndroid領域をHalium-13 LXCとlibhybrisが利用する構成です。boot.imgを一時起動するだけで、init_boot/vendor_kernel_boot/rootfsまでV1の組み合わせに切り替わるとは扱いません。Buildrootの90秒復帰処理もV1には組み込まれていません。

## Aurora UIの土台にする方針

既存のAsteroidOSの表示・入力・Qt 6構成を基準として、まず同じ実機での時計画面とタッチの動作実績を確認します。その後に独自Aurora UIを載せる方針です。V1のRelease説明には「Wi-Fiは使用可能、常時表示は不可」とありますが、今回の静的解析で再検証した結果ではありません。

引き継ぐ条件は、管理者権限や追加ドライバを要求しない物理USBシェル等の確保と、スピーカーを無効にすることです。V1は `CONFIG_SOUND=y`、音声モジュール自動読み込み、PulseAudioユーザーサービス有効という構成なので、そのままで消音要件を満たすとはしません。

Buildroot側で実機確認したUSBシェル・復帰方法・DT情報・不具合切り分けは調査資料として保持します。Buildrootの5.15.137用モジュールをV1の5.15.144へ混在させたり、ARM32というだけでmuslとglibcの動的リンクバイナリを互換扱いしたりしません。カーネル修正もV1のソースへ適用可能か個別に確認します。

本PRは移行の基準点を記録するものです。書き込み用コマンド、パーティション変更、新しい配布イメージは含みません。
