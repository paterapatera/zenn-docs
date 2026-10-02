---
title: "JavaScriptとVOSKで自動文字起こし録音ツールを作ってみた"
emoji: "🎤"
type: "tech"
topics:
  - "javascript"
  - "音声認識"
  - "文字起こし"
  - "vosk"
published: true
published_at: "2023-08-21 12:00"
---

マイクで拾った音を文字起こしして、その部分の音声をMP3にするツールを作りました。
そしてHTMLとして出力するので、ブラウザから画像のように確認することができます。
![文字起こしツールの画像](https://storage.googleapis.com/zenn-user-upload/376151a08fd5-20230811.png)
*文字起こしに使用した動画：【ストレスに勝つ】強いメンタルをつくる脳の鍛え方！*

# コードの説明

このツールはJavaScriptとVOSKという音声認識のライブラリで作りました。

ソースコードはこんな感じです。
```js
const memo = new Memo() // HTMLとしてファイルに書き込むためのクラス
const mp3 = new Mp3(dayjs()) // MP3ファイルとして書き込むためのクラス
mp3.rotate(memo, dayjs())

setLogLevel(-1)
const model = new Model(MODEL_PATH)
const rec = new Recognizer({ model: model, sampleRate: SAMPLE_RATE })

// 文字起こしプロセス
const micInstance = mic({
  rate: String(SAMPLE_RATE),
  channels: '1',
  debug: false,
  device: 'default',
})

const micInputStream = micInstance.getAudioStream()

micInputStream.on('data', (data) => {
  if (rec.acceptWaveform(data)) {
    const result = rec.result()
    if (Memo.isExist(result)) {
      memo.writeMessage(result, dayjs())
      mp3.rotate(memo, dayjs())
    }
  }
})

micInputStream.on('audioProcessExitComplete', function () {
  memo.writeMessage(rec.finalResult(), dayjs())
  memo.end()
  rec.free()
  model.free()
})

// 録音プロセス
const recInstance = mic({
  rate: String(SAMPLE_RATE),
  channels: '1',
  debug: false,
  device: 'default',
  fileType: 'mp3',
})

const recInputStream = recInstance.getAudioStream()

recInputStream.on('data', (data) => {
  mp3.write(data)
})

recInputStream.on('audioProcessExitComplete', function () {
  mp3.end()
})

// プロセスイベント
process.on('SIGINT', function () {
  micInstance.stop()
  recInstance.stop()
})

// 開始
micInstance.start()
recInstance.start()
```
コード全体は[こちら](https://github.com/paterapatera/gijiroku-rec/blob/main/index.mjs)

内容を大きく分けると、音声認識と録音のプロセスがあります。

## 音声認識プロセスの説明

音声認識プロセスの主要部分を説明します。
下記部分で取得した音声データを取得したときに、データを`rec.acceptWaveform(data)`へと渡すようにしています。
```js
micInputStream.on('data', (data) => {
  if (rec.acceptWaveform(data)) {
    const result = rec.result()
    if (Memo.isExist(result)) {
      memo.writeMessage(result, dayjs())
      mp3.rotate(memo, dayjs())
    }
  }
})
```

このイベントは、音声が一定間隔のぶつ切りのデータで渡されます。
たとえば「こんにちは」とマイクに話した場合

1. 「こん」でイベントを実行
2. 「にち」でイベントを実行
3. 「は」でイベントを実行

という感じになります。
なので、「こんにちは」という言葉の区切りに達したかを`rec.acceptWaveform(data)`で判定しています。
そして、`rec.acceptWaveform(data)`は、音声データのストックも同時に行っているようです。

ざっくりとした一連の流れは

1. 喋ってる途中なら、`rec.acceptWaveform(data)`はfalseで、以降の処理は実行されない
2. 喋り終わったら、`rec.acceptWaveform(data)`がtrueになる
3. 「こんにちは」の音声データがストックされているので、文字起こしの結果を`rec.result()`から取得する
4. 取得結果を`memo.writeMessage()`でファイルに出力する

`mp3.rotate()`は現在のMP3ファイルへの録音を終了して、新しいMP3ファイルにて録音を再開する処理です。

## 録音プロセスの説明

録音プロセスも音声認識と内容はあまり変わりません。
```js
const recInstance = mic({
  rate: String(SAMPLE_RATE),
  channels: '1',
  debug: false,
  device: 'default',
  fileType: 'mp3',
})

recInputStream.on('data', (data) => {
  mp3.write(data)
})
```
違いはインスタンス作成時に`fileType: 'mp3'`を追加しています。
これによりmp3に変換したデータを受け取ることができます。

その他は、マイク音取得イベントでmp3データを取得して、それをmp3ファイルへ書き込んでいるぐらいです。

## 注意点

```js
process.on('SIGINT', function () {
  micInstance.stop()
  recInstance.stop()
})
```
この処理を書き忘れないように注意する必要があります。
`stop()`で必ず止めるようにしないと、ツールが動き続けてctrl+cで停止できなくなってしまいます。

# ツールの使い方

実際にこのツールを動かすには以下の準備が必要です

1. VOSKの日本語モデルデータをhttps://alphacephei.com/vosk/modelsからダウンロードしてくる
2. コマンドラインでマイクを使うために、Macの場合`brew install sox`が必要
3. PC内の音をマイクとして使いたいなら、その設定が必要
   - 私はMacでSoundflowerとAudio MIDI設定(既存)の対応をしました。

準備が終われば、以下のように保存先を指定して実行できます。
```sh
node index.mjs ./test
```
音声ファイルは2時間で50MBぐらいでした。

# 作った経緯

なぜこのツールを作ろうと思ったかですが、仕事で議事録を書くことが多くなってきたからです。
リーズナブルに文字起こしできるサービスを探していたんですが、なかなか条件にあうものが見つかりませんでした。
こうなったら自分で作るしかないと思ったわけです。

# 使ってみた感想
文字起こしの精度は85%程度でした。
たまに意味不明な部分もあったりして、これだけだと、そこまで使い勝手が良くなかったです。
しかし、録音音声を聞けるようにしたことでかなり使いやすくなりました。

今回はそんな感じで、とてもいい勉強になりました。