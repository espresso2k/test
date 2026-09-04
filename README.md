# DJI PSDK Liveview を実行可能な dpk パッケージにする手順メモ

> 対象: Manifold 環境で DJI Payload SDK (PSDK) の Liveview sample をクロスコンパイルし、`dpk` としてインストール可能にするまで。

## 0. 事前に押さえる全体像

1. **PSDKソース取得**（サブモジュール含む）
2. **クロスコンパイルツールチェーン準備**（対象SoCとABIを一致）
3. **Liveview sample をビルド**（依存ライブラリのリンク確認）
4. **実行物 + 設定 + サービス定義を dpk 構成に配置**
5. **dpk パッケージ生成**
6. **Manifoldへ転送・インストール・起動確認**

---

## 1. ソース取得と初期化

```bash
git clone https://github.com/dji-sdk/Payload-SDK.git
cd Payload-SDK
git submodule update --init --recursive
```

- `--recursive` を忘れると、後段のビルドでサブモジュール欠落エラーになりやすい。

---

## 2. クロスコンパイル環境を固定する

DJI公式ドキュメント側の **対象プラットフォーム（Manifold世代）に対応する gcc / sysroot** を必ず合わせる。

### 2.1 ツールチェーン確認項目

- `CMAKE_TOOLCHAIN_FILE` が対象向けか
- `CMAKE_SYSROOT` が実機ルートFS由来か
- `CC/CXX` と `ar/ranlib/strip` が同一プレフィックスか
- `readelf -h` で出力ELFの `Machine` が期待通りか

### 2.2 典型的な CMake 例（概念）

```bash
mkdir -p build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE=../cmake/toolchain-xxx.cmake \
  -DPSDK_PLATFORM=manifold
make -j$(nproc)
```

> 実際のオプション名・toolchainファイル名は、使うPSDKバージョンに合わせて読み替える。

---

## 3. Liveview sample のビルドと動作確認

- まずは `sample_c`（または該当sample）だけを確実に通す。
- 実機投入前に `file <binary>` でアーキ確認。
- 可能なら `ldd` 相当（クロス環境では `readelf -d`）で動的依存を確認。

**最低限の確認ポイント**

- バイナリが ARM 向けになっている
- `libstdc++`, `libpthread`, `libm`, `librt` など必要soが揃う
- 実機側ランタイムのバージョンとズレていない

---

## 4. dpk パッケージ化の進め方（実務向け）

`dpk` は最終的に「**配置先ディレクトリ構造 + メタ情報 + 起動方法**」を固める作業。

### 4.1 パッケージディレクトリ例

```text
pkg/
  manifest.json            # パッケージ情報
  bin/liveview_sample      # 実行ファイル
  lib/*.so                 # 同梱が必要な共有ライブラリ
  conf/*.json              # PSDK設定
  service/*.service        # 自動起動させる場合
  scripts/postinst.sh      # インストール後処理（必要時）
```

### 4.2 作成時の要点

- 実行ファイルの `chmod +x` を忘れない。
- `RPATH` 設計を先に決める（`$ORIGIN/../lib` など）。
- 依存soを同梱するか、実機標準に寄せるかを明確化。
- 自動起動するならサービス定義とログ出力先を先に作る。

### 4.3 dpk 生成

Manifold SDK側の `dpk` 作成コマンド（またはスクリプト）で `pkg/` を固める。

```bash
# 例: 実際はManifold側ツールの仕様に合わせる
dpk pack ./pkg -o liveview_sample.dpk
```

---

## 5. 実機導入

```bash
scp liveview_sample.dpk user@manifold:/tmp/
ssh user@manifold
sudo dpk install /tmp/liveview_sample.dpk
sudo dpk list | grep liveview
sudo systemctl status <service-name>
```

- 起動直後はログ監視を優先。
- Liveview は帯域・遅延の影響を受けるため、CPU使用率とネットワーク状態も同時確認。

---

## 6. クロスコンパイルで躓きがちな点（重要）

1. **ABI不一致（最頻出）**
   - `gnueabihf` と `gnu` の取り違え、`hard-float`/`softfp` 不整合。
   - 症状: 実機で `Exec format error` / 起動時クラッシュ。

2. **sysroot が古い or 実機と別物**
   - ビルドは通るが実機で `GLIBCXX_x.x.x not found`。

3. **ホスト側ライブラリを誤リンク**
   - クロスビルド時に `/usr/lib/x86_64-linux-gnu` を拾ってしまう。
   - 対策: `CMAKE_FIND_ROOT_PATH_MODE_*` を target 優先に固定。

4. **サブモジュール未取得**
   - `git submodule update --init --recursive` 忘れ。

5. **OpenSSL / curl / ffmpeg 系の依存解決ミス**
   - Liveview周辺はマルチメディア依存が出やすい。
   - バージョン固定、`pkg-config` の `PKG_CONFIG_SYSROOT_DIR` を明示。

6. **RPATH未設定で起動時にsoが見つからない**
   - 対策: `-Wl,-rpath,'$ORIGIN/../lib'` を使用。

7. **strip のやりすぎ**
   - 必要シンボルまで落として不具合解析不能。
   - `objcopy --only-keep-debug` でデバッグシンボル別保管推奨。

8. **権限・サービス連携不足**
   - dpk インストール後に実行権限不足、service unit の `WorkingDirectory` 不整合。

9. **時刻同期・証明書関連**
   - SSL/TLS通信を使う場合、時刻ズレで接続失敗。

10. **ログ設計不足**
    - 問題切り分け時に `journalctl` だけでは情報不足。
    - 起動引数・SDK初期化・ストリーム確立の各フェーズでログを出す。

---

## 7. 実務での最短デバッグ順

1. `file`, `readelf -h`, `readelf -d` でバイナリ整合を確認
2. 実機で最小起動（サービス化せず手動実行）
3. 必要so不足を解消
4. その後に service 化
5. 最後に dpk 化して再現性を固める

---

## 8. 参考

- DJI Manifold 開発クイックスタート
  - https://developer.dji.com/doc/payload-sdk-tutorial/en/manifold-quick-start/manifold-devlopment.html
- DJI Payload-SDK リポジトリ
  - https://github.com/dji-sdk/Payload-SDK

