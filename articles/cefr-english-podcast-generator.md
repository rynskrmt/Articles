---
title: "理想の英語学習ツールを自分で開発してみた - OpenAI TTS API, Python(Gradio)🚀"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## はじめに：パーソナライズされた英語リスニング教材の必要性

英語学習、特にリスニング力の向上において、自身の興味やレベルに合致した教材を見つけることは重要な要素です。しかし、市販の教材やオンラインサービスでは、特定のトピックや細かなレベル調整に対応しきれないケースが多くありました。

私自身、特定の専門分野に関する議論を、自身の英語レベルに適した難易度で聞きたいと考えていましたが、既存の教材ではなかなか見つけることができませんでした。

この課題を解決するため、「**自分の興味・レベルに合わせて、リスニング教材を生成できるツール**」の必要性を感じ、開発に至りました。本記事では、その開発経緯、技術的な実装について共有します。

## 課題設定：理想的な英語リスニング教材の要件

開発にあたり、理想的なリスニング教材に必要な要件を以下のように定義しました。

1.  **トピックの自由度:** ユーザーが任意のキーワードを指定し、それに基づいた内容が生成されること。
2.  **レベルの最適化:** CEFR基準（A1〜C2）で難易度を指定でき、語彙や表現が調整されること。
3.  **自然な会話形式:** 複数の話者による、リアルな対話形式（Podcast風）であること。
4.  **高品質な音声:** クリアで自然な発音の音声ファイルが提供されること。
5.  **オンデマンド生成:** 必要な時にすぐに新しい教材を生成できること。

これらの要件を既存のソリューションで満たすことは難しく、特にニッチなトピックとレベル調整の両立が技術的な課題でした。

## 開発したもの：「CEFR English Podcast Generator」

上記の課題を解決するために開発したのが、**「CEFR English Podcast Generator」**です。

これは、OpenAI API (GPT-4oによるテキスト生成、TTSによる音声合成) と Gradio (UIフレームワーク) を組み合わせたPythonアプリケーションです。

**主な機能:**

*   ユーザーが指定した**トピック、CEFRレベル、単語数、追加情報**に基づき、AIが**英会話スクリプト**を生成。
*   生成されたスクリプトを元に、**MP3形式の音声ファイル**を自動生成。

これにより、定義した要件を満たすパーソナライズされたリスニング教材のオンデマンド生成が可能になりました。

## 実装の詳細：Python と OpenAI API による実現方法

アプリケーションのコア機能はPythonで実装し、主要な処理はOpenAI APIを利用しています。

*   **言語:** Python 3.x
*   **AI API:** OpenAI API
    *   テキスト生成: `gpt-4o` (環境変数 `TEXT_MODEL` で指定)
    *   音声合成: `tts-1` (環境変数 `AUDIO_MODEL` で指定)
*   **Web UI フレームワーク:** Gradio
*   **ライブラリ:** `python-dotenv`, `openai`

### システム構成と処理フロー

全体の処理フローは以下の通りです。
![](/images/cefr-english-podcast-generator/diagram2.png)

### 1. 会話テキスト生成 (`generate_dialogue`)

中核となるのが、GPTモデルに対するプロンプト設計です。意図した通りの対話形式、内容、難易度、長さを実現するために、詳細な指示を与えています。

```python
# プロンプトの主要部分
prompt = f"""
You are tasked with generating an engaging, educational and fun podcast dialogue designed specifically for English language learners at CEFR level {cefr_level}.
The conversation should focus on the topic: '{topic}'. {additional_info_text}
Craft a natural, lively, and informative discussion between two speakers ({SPEAKER_1_NAME} and {SPEAKER_2_NAME})...
IMPORTANT: The dialogue MUST be EXACTLY {word_count} words in total length. This is a strict requirement.
...
Your dialogue must:
    - Be EXACTLY {word_count} words total. Count carefully.
    - Clearly alternate turns between {SPEAKER_1_NAME} and {SPEAKER_2_NAME}.
    - Avoid special characters or markdown formatting.
    - Define any specialized terms clearly and simply...
    - Include a few jokes and interesting facts.
    - Have a clear introduction, body with multiple discussion points, and conclusion.
Use the following format:
    {SPEAKER_1_NAME}: sentence
    {SPEAKER_2_NAME}: sentence
...
"""

# API呼び出し
response = client.chat.completions.create(
    model=TEXT_MODEL,
    messages=[{"role": "user", "content": prompt}],
    max_completion_tokens=max_tokens, # 単語数から推定
    temperature=0.7, # ある程度の多様性を許容
)
dialogue_content = response.choices[0].message.content

# コスト計算
calculate_text_cost(response, TEXT_MODEL)
```

### 2. テキストから音声への変換 (`text_to_audio`, `get_mp3`)

生成されたテキストは、改行で区切られた各発話ごとにTTS APIへ送信されます。

```python
def text_to_audio(text, speaker_1_voice, speaker_2_voice, audio_model):
    # ... (一時ファイル、ログの準備) ...
    dialogue_lines = text.split('\n')
    with open(file_path, 'wb') as f:
        for line_idx, line in enumerate(dialogue_lines):
            # スピーカー判定と内容抽出
            if line.startswith(f"{SPEAKER_1_NAME}:"):
                voice = speaker_1_voice
                content = line.replace(f"{SPEAKER_1_NAME}:", "").strip()
            elif line.startswith(f"{SPEAKER_2_NAME}:"):
                voice = speaker_2_voice
                content = line.replace(f"{SPEAKER_2_NAME}:", "").strip()
            else:
                continue # スキップ

            if not content: continue # スキップ

            try:
                # ストリーミングAPIを利用
                with client.audio.speech.with_streaming_response.create(
                    model=audio_model, voice=voice, input=content,
                ) as response:
                    # 取得したオーディオチャンクをファイルに追記
                    for chunk in response.iter_bytes():
                        f.write(chunk)
                # TTSコスト計算
                calculate_tts_cost(content, audio_model)
            except Exception as e:
                # エラーログ記録 (リトライ処理も検討可能)
                log_error(f"TTS Error on line {line_idx+1}: {e}", content)
    return file_path

def get_mp3(text, voice, audio_model): # text_to_audioから呼び出される想定
    # (calculate_tts_cost, client.audio.speech... の実装)
    pass
```

**実装のポイント:**

*   **発話単位での処理:** テキスト全体を一度にTTSにかけるのではなく、各スピーカーの発話を個別に処理。これにより、異なるボイス（`alloy`, `echo`等）を割り当て可能に。
*   **ストリーミング応答の利用:** `with_streaming_response` を用いることで、APIからの音声データをチャンクで受け取り、メモリ消費を抑えつつファイルに書き込めます。長い会話でも効率的に処理可能です。
*   **バイト列の結合:** 各発話から得られたMP3データのバイト列を単純にファイルに追記していくことで、最終的な一つの音声ファイルを生成しています。
*   **エラーハンドリングとログ:** TTS APIはネットワークや入力内容によってエラーを返す可能性があるため、`try-except` による例外処理と、問題発生箇所を特定するためのログ出力が重要です。

### 3. GradioによるシンプルなUI構築

ユーザーインターフェースにはGradioを採用しました。Pythonコードのみで対話的なWeb UIを迅速に構築できる点がメリットです。

```python
import gradio as gr

# ... (generate_audio_dialogue 関数の定義) ...

ui = gr.Interface(
    fn=generate_audio_dialogue,
    inputs=[
        gr.Textbox(label="トピック", placeholder="e.g., Indie Hacking"),
        gr.Textbox(label="追加情報（オプション）", lines=3),
        gr.Dropdown(choices=["A1", "A2", "B1", "B2", "C1", "C2"], label="CEFRレベル", value="B1"),
        gr.Slider(minimum=100, maximum=5000, value=300, step=50, label="単語数"),
    ],
    outputs=[
        gr.Textbox(label="トランスクリプト", lines=15), # linesで高さを指定
        gr.Audio(label="音声ファイル", type="filepath"),
        gr.Textbox(label="API利用料金"),
    ],
    title="CEFR English Podcast ジェネレーター",
    description="トピックとCEFRレベルに基づいて英語学習者向け会話音声教材を生成します。"
)

if __name__ == "__main__":
    ui.launch()
```
Gradioの`Interface`クラスに必要な関数と入出力コンポーネントを指定するだけで、基本的なUIが完成します。

## 導入と利用方法

以下の手順でローカル環境で実行できます。

1.  **リポジトリのクローン:**
    ```bash
    git clone https://github.com/rynskrmt/CEFR-English-Podcast-Generator.git
    cd CEFR-English-Podcast-Generator
    ```
2.  **依存パッケージのインストール:**
    ```bash
    pip install -r requirements.txt
    ```
3.  **環境変数の設定:**
    ルートディレクトリに`.env`ファイルを作成し、OpenAI APIキーを記述します。必要に応じてモデル名なども変更可能です。
    ```dotenv
    OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    TEXT_MODEL=gpt-4o
    AUDIO_MODEL=tts-1
    SPEAKER_1_VOICE=alloy
    SPEAKER_2_VOICE=echo
    SPEAKER_1_NAME="Speaker 1"
    SPEAKER_2_NAME="Speaker 2"
    ```
4.  **アプリケーションの起動:**
    ```bash
    python app.py
    ```
5.  **ブラウザでアクセス:**
    ターミナルに表示されるローカルURL（例: `http://127.0.0.1:7860`）を開き、インターフェースから操作します。

## まとめ

本稿では、パーソナライズされた英語リスニング教材をオンデマンドで生成するツール「CEFR English Podcast Generator」の開発について紹介しました。OpenAI APIとGradioを活用することで、当初抱えていた「自分に合った教材が見つからない」という課題を効果的に解決できました。

**ソースコードはこちらで公開しています:**
https://github.com/rynskrmt/CEFR-English-Podcast-Generator

フィードバックや改善提案など、お気軽にIssueやPull Requestをお寄せください。
