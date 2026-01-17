# 🍎 GPT-SoVITS for MacBook Pro (M5) Optimized Manual

このドキュメントは、Apple Silicon (M-series) 環境で GPT-SoVITS を安定動作させ、かつ研究・制作環境をクリーンに保つための完全ガイドです。

---

## 🛠 0. 環境構築と不足ライブラリの解消

標準のインストールだけでは、Mac環境での多言語処理や音声読み込みでエラーが発生します。以下の手順で不足分を補完します。

### 1. 依存ライブラリの追加インストール

Macの音声処理エンジン（CoreAudio）と `torchaudio` の相性問題を解決するため、および多言語判定のために以下のライブラリを手動で追加しています。

```bash
# uv環境下での実行
uv pip install librosa soundfile nltk psutil fast-langdetect

```

### 2. NLTK（自然言語処理）データの事前取得

英語ナレーション生成時に「MacBook」や「AI」などの単語でシステムが止まるのを防ぐため、以下のトークナイザーを強制ダウンロードします。

```bash
uv run python -c "import nltk; nltk.download('averaged_perceptron_tagger_eng'); nltk.download('averaged_perceptron_tagger'); nltk.download('punkt'); nltk.download('punkt_tab')"

```

---

## 📝 1. コード修正の全記録（Mac専用パッチ）

標準コードの `torchaudio` は、Macにおいて `libtorchcodec` 関連のランタイムエラーを引き起こします。これを回避するため、**汎用性の高い `librosa` へ置換**する修正を行いました。

### ① 学習データ生成部

**ファイル:** `GPT_SoVITS/prepare_datasets/2-get-sv.py`

* **修正内容:** 音声特徴量抽出時の読み込みエンジンを変更。
* **コード:**
```python
import librosa  # 追加
import torch

# 修正前: audio, sr0 = torchaudio.load(file_path)
# 修正後:
audio, sr = librosa.load(file_path, sr=None)
audio = torch.from_numpy(audio).unsqueeze(0) # Tensor型へ変換して後続処理へ渡す

```



### ② 推論（音声生成）WebUI部

**ファイル:** `GPT_SoVITS/inference_webui.py`

* **修正内容:** 524行目付近の `get_spepc` 関数を修正。参照音声の読み込み失敗を防止。
* **コード:**
```python
import librosa # 先頭に追加

def get_spepc(hps, filename, dtype, device, is_v2pro=False):
    # 修正箇所
    audio, sr0 = librosa.load(filename, sr=None)
    audio = torch.FloatTensor(audio).unsqueeze(0)
    # 以降、オリジナル処理へ継続

```



---

## 🧹 2. ルートディレクトリのクリーンアップ戦略

新しい音声を追加するたびに生成される不要なバージョンフォルダ（v3, v4等）を整理し、視認性を高めます。

### 1. `webui.py` のバージョン固定

`webui.py` の 4行目付近で、使用する学習エンジンを最新の `v2Pro` に固定しています。

```python
os.environ["version"] = version = "v2Pro"

```

### 2. 自動掃除スクリプト（推奨）

ルートディレクトリを汚さないよう、以下の内容を `clean_env.sh` として保存して実行することを推奨します。

```bash
#!/bin/bash
# 必要な v2Pro 関連以外をすべて 'unused_files' に退避
mkdir -p unused_files
mv GPT_weights GPT_weights_v2 GPT_weights_v3 GPT_weights_v4 unused_files/ 2>/dev/null
mv SoVITS_weights SoVITS_weights_v2 SoVITS_weights_v3 SoVITS_weights_v4 unused_files/ 2>/dev/null
mv *.bat *.ps1 Docker Dockerfile unused_files/ 2>/dev/null
echo "Cleanup complete. Target version: v2Pro"

```

---

## 🔄 3. 外部音声（Desktop等）を使用した実運用フロー

ファイルをプロジェクト内にコピーせず、外部パスのまま処理する「Kazukiスタイル」の全手順です。

### Step 0: プログラム起動

```bash
cd /Users/kazuki/Desktop/GPT-SoVITS
uv run python webui.py

```

### Step 1: ノイズ除去（0a-UVR5）

* **入力パス:** `/Users/kazuki/Desktop/voice/voice.wav`
* ※ `⌥ (Option)` + `右クリック` でパスをコピーすること。


* **出力:** `output/uvr5_opt/` に自動生成。

### Step 2: データセット作成（1Aタブ）

1. **実験名:** `voice_project_01`（必ず新しい名前を付ける）。
2. **スライス/ASRパス:** `output/uvr5_opt` を指定。
* ※ `output/slicer_opt` や `output/asr_opt` に実験名ごとのサブフォルダが作られるため、管理が容易になります。



### Step 3: 学習（1Bタブ）

* SoVITS/GPTの両ボタンをクリック。
* **成果物:** `GPT_weights_v2Pro/voice_project_01-e15.ckpt` 等が自動生成。

### Step 4: 推論（1Cタブ）

1. モデルのパスを更新し、`voice_project_01` を選択。
2. 推論WebUI（9872）を起動。
3. **参照音声:** `output/slicer_opt/voice_project_01/xxx.wav` から良さそうなものを1つ選ぶ（これが最も本人に似るコツです）。

---

## 💡 運用上のヒント

* **M5 Macのパフォーマンス:** 学習時の「バッチサイズ」は `12`〜`16` 程度が安定します。
* **音質の秘訣:** 「0b-スライス」の際、あまり短く切りすぎないこと（5〜8秒が理想）。

---

これで、技術的な背景（なぜライブラリを足したか、どこを書き換えたか）を含めた完璧なマニュアルになりました。これを `README.md` としてGitHubにアップすれば、他のMacユーザーにとっても最高のバイブルになりますね。

次は、このGitHubリポジトリに含めるための `.gitignore`（不要な巨大ファイルをアップしない設定）の作成もお手伝いしましょうか？