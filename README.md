# postmortem

過去に葬り去ったバグ

## Base64

メモリレイアウトをそのままBase64として出力するコードをRustで書いていた際、[`bytemuck::Pod`](https://docs.rs/bytemuck/1.14.0/bytemuck/trait.Pod.html)トレイトを実装した型がBase64として直接シリアライズできると考えていた。\
この仮定は間違ってはいないものの、参照を渡したときにそのアドレスが直接出力されてしまうという不具合があった。\
そこで、参照については逆参照した内部の値を書き込むように行うことで修正を図った。

## VRChatRejoinTool

[VRChatRejoinTool](https://github.com/yanorei32/VRChatRejoinTool)という、VRChatにおいて本来明示的な招待がなければ再入することができない[^1]インスタンス[^2]に再入するためのツールがある。\
そのVRChatが吐くログのパーサー部分で条件変数を[逆にする](https://github.com/yanorei32/VRChatRejoinTool/commit/16af44eaf99cc84ca47121fcf185f3990c640569)という凡ミスを犯した。

[^1]: VRChatの仕様により、インスタンスに入るためのURLが判明していれば招待を迂回することができる。
[^2]: Discordで言うチャンネルのようなもの。元となるワールドを`class`指向プログラミングよろしくインスタンス化することでインスタンスが生成される。誰もいなくなると廃棄される。

## origlang

[origlang]は0からRustでプログラミングされた自作言語処理系を指す。

[origlang]: https://github.com/KisaragiEffective/origlang

### UTF-8処理における問題
[origlang]は入力文字列として**常に**validなUTF-8を受け取るので、常に`char`の境界バイトであることが保証されている現在位置から`char`がデコードできる。そこで、現在位置から数バイト読むことで`char`をデコードする実装を
[書いた](https://github.com/KisaragiEffective/origlang/commit/49ea8077c1b995b4018cc71106a8637580ede287#diff-0277ac4bb2f811784bb9e362932e3d7270f8124fa5f68aba990d22db57edda0bR473)。
しかし、このコードにはバグが複数あった。
1. UTF-8の先頭バイトのマスクが正しくない \
    ASCII範囲外の先頭バイトの上位ビットは1が連続している。このビット長は2以上4以下であり、そのままそのコードポイントを表現するのに必要なバイト長に対応する。\
    それを取り出すために上位ビットに対して`0b11000000`のようなビットマスクをビット積の右辺に渡していた。\
    しかし、ビットマスクが1ビット少なかったため本来3バイト文字を表現しているコードポイントが4バイトとして扱われるなど不具合が生じた。
2. 2バイトを意味する先頭バイトを4バイトとして扱っている\
    上記1のバグを組み入れたことで、2バイトのコードポイントの先頭バイトのビットマスクが`0b10000000`となっていた。そして2バイト文字として扱われるはずだった。\
    しかし、コピーアンドペーストによるミスで4バイトのコードポイントの先頭バイトとして扱われる[実装](https://github.com/KisaragiEffective/origlang/blame/b08fbbf4351e855fa1a93b477e0e43989b8e0160/package/origlang-compiler/src/lexer.rs#L583)になっていた。

その結果、おそらくではあるが意図しない異常終了が起きていたものと考えられる。

対策として、修正に加え単純な[単体テスト](https://github.com/KisaragiEffective/origlang/pull/244/files#diff-2db0347bcbc239369aaa80eaca96922d14f91cf3bd414335e99d028e7e9b31eeR97)を追加した。

### リファクタリングによるリグレッション
[origlang]は入力文字列として**常に**validなUTF-8[^3]を受け取るので、Unicodeのコードポイントである32ビット幅の[`char`](https://doc.rust-lang.org/std/primitive.char.html)にデコードすることなくパーサーの位置を進めるためのリファクタリングを行った。\
そのリファクタリングで数値リテラルのコードポイントを範囲で指定する際、本来であれば`9`を含めるべきだった。
しかし、他の言語と勘違いしたためか`9`を含めなかった[^4]ため`9`を文字列に含む整数リテラルがパースできないという[バグ](https://github.com/KisaragiEffective/origlang/issues/291)が発生した。\
また、Rustの公式linterであるClippyによって[`almost_complete_range`](https://rust-lang.github.io/rust-clippy/master/#/almost_complete_range)が発出されていたはずだが、当時はCIによるClippy実行を行っていなかったため検知できなかった。\
さらに、パーサーには単体テストがあったものの、テストでチェックされている値は`1`、`2`、`123456`のみだった。

対策として、[単体テストの追加](https://github.com/KisaragiEffective/origlang/pull/290/files#diff-2db0347bcbc239369aaa80eaca96922d14f91cf3bd414335e99d028e7e9b31ee)及び[CIによるClippyの強制](https://github.com/KisaragiEffective/origlang/pull/293)を行った。

## findコマンドで無限ループを招いた罪

```sh
find . -type d -exec mkdir -p ./Hoge/{} \;
```

このコマンドを使うと、おそらく以下のような作用機序で無限ループを招いて死ぬ。

1. Piyoフォルダの下にHogeを作る
2. Piyo/Hogeは探索されていないので、Piyo/Hogeの下にHogeを作る
3. Piyo/Hoge/Hogeは……(以下、無限ループ)

対策として、一旦パイプに逃がすことで解決した。

```sh
find . -type d -print0 | xargs -0 -I {} mkdir -p "./Hoge/{}"
```

[^3]: Rustの`str`は常にUTF-8であることが保証されており、`&mut [u8]`などで可変借用した結果その不変条件が破れていた場合は未定義動作となる。
[^4]: 例えば、Rubyは`"0".."9"`とすると`"9"`を含む。
