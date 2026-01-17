# RVC-Boss GPT-SoVITS for Mac 最適化ガイド

**対象:** macOS (Apple Silicon M1/M2/M3/M5...)  
**環境:** `uv` パッケージマネージャー使用時  
**目的:** 自分の音声（日本語）を英語・中国語などの音声に変換するポストプロセス

---

## 1. 環境構築（インストール）

Macでは、標準の `pip` よりも高速で依存関係を管理しやすい `uv` を使用します。

### 1-1. FFmpegのインストール（必須）
MacのターミナルでFFmpegが入っていないと、音声処理でコケます。
```bash
brew install ffmpeg
```

### 1-2. リポジトリのクローンと環境構築
```bash
cd Desktop
git clone https://github.com/RVC-Boss/GPT-SoVITS.git
cd GPT-SoVITS
```

### 1-3. 仮想環境とライブラリのインストール
今回は `uv` を使います。**`librosa` と `soundfile` を必ず入れる**のがMac安定化のコツです。

```bash
# 仮想環境作成（既にある場合はスキップ）
uv venv

# ライブラリインストール
# ※ requirements.txtに含まれていないが、Macでは必須のものを追加しています
uv pip install -r requirements.txt
uv pip install librosa soundfile numpy
```

---

## 2. Mac環境での重要な修正（コードパッチ）

PyTorchの最新バージョン（2.9系）とMacの組み合わせでは、標準の `torchaudio.load` がエラー（`RuntimeError: Could not load libtorchcodec`）を吐くことがあります。これを回避するために、**読み込み処理を `librosa` に置き換える**のが最も安定します。

### 対象ファイル
`GPT_SoVITS/prepare_datasets/2-get-sv.py`

### 修正手順
1.  ファイルを開く。
2.  **インポート部分の追加**（一番上）
    ```python
    import librosa
    ```

3.  **読み込み関数の書き換え**（92行目あたり、`def name2go` 内）
    **修正前:**
    ```python
    wav32k, sr0 = torchaudio.load(wav_path)
    ```

    **修正後:**
    ```python
    # librosaを使って読み込む（Macのエラー回避）
    wav, sr0 = librosa.load(wav_path, sr=None)
    wav32k = torch.from_numpy(wav).unsqueeze(0)
    ```

---

## 3. 実行と運用のポイント

Macメモリ管理のため、ターミナルは**「サーバー用」「作業用」の2つ立ち上げ**るのがおすすめです。

### 3-1. サーバー起動（ターミナル1）
ずっと動かしっぱなしにします。
```bash
uv run python webui.py
```
ブラウザで `http://0.0.0.0:9874` を開きます。

### 3-2. 推奨されるワークフロー
WebUI上で作業を進めます。

#### Step 1: データ準備（タブ 0）
1.  音声ファイル（`my_voice.wav` など）と、テキスト（台本）をアップロードします。
2.  「0A - 音声の自動切り出し」などのボタンを押して処理します。

#### Step 2: 特徴抽出（重要！）
ここでMacユーザーがよく詰まります。
**「一番下にある「一括処理（Batch）」ボタンは使わず、個別のボタンを押す**方がエラー箇所の特定ができ、安定します。

*   **「1Aa - テキスト分割と特徴抽出」** のボタンを押す。
*   **「1Ab - 音声自己教師あり特徴抽出」** のボタンを押す。
    *   ここで `librosa` の修正が効いてきます。
    *   ターミナルに進捗（%）が出るのを確認し、終わるまで待ちます。

#### Step 3: 学習（トレーニング）
*   **「1Ac - セマンティックトークン抽出」** を実行。
*   **「1B - SoVITSトレーニング」** ボタンを押して、モデルを作成します。
    *   M5チップなら10〜15分程度で終わります。

---

## 4. ゴール：音声変換の使い方（タブ 1）

学習が完了したら、いよいよ変換です。

1.  WebUI上部のタブで **「1 - GPT-SoVITS-TTS」** をクリック（※「2」はリアルタイム用）。
2.  **モデル選択:** 学習させた自分のモデルが選択されているか確認。
3.  **言語設定:** 変換したい言語（英語/中国語）を選択。
4.  **テキスト入力:** ポッドキャストの台本（英語や中国語の文章）を入力。
5.  **生成ボタン:** クリックすると、あなたの声で喋る音声が生成されます。

---

## 5. トラブルシューティング

### Q: 進捗バーが止まったまま動かない（10分以上）。
A: たぶんフリーズしています。
*   ターミナルで `Control + C` して止める。
*   `uv pip install soundfile` が実行されているか確認する。
*   個別のボタン（1Aa, 1Ab）を1つずつ順番に試す。

### Q: `FileNotFoundError: slice_opt.list` が出る。
A: ASR（文字起こし）の出力ファイル名ミスマッチです。
*   Finderで `output/asr_opt/` 内の `slicer_opt.list` の名前を `slice_opt.list` に変更してください。

### Q: 生成した音声が遅い（再生速度）。
A: Mac（CPU）での処理は時間がかかります。出力された音声ファイルをダウンロードして、DAWや再生ソフトで聞けば正常速度で再生されます。

---

**Happy Voice Conversion!**