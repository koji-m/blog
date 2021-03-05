---
title: "Deeplearning Framework in Ruby"
description: "書籍[ゼロから作る Deep Learning ❸](https://github.com/oreilly-japan/deep-learning-from-scratch-3)で題材となっているDeep Learningフレームワーク DeZeroをRubyと[Numo::NArray](https://github.com/ruby-numo/numo-narray)を使って実装してみたので、現状できることの紹介と実装時のポイントをいくつか説明します。"
date: 2021-03-05T20:59:14+09:00
draft: true
---

書籍[ゼロから作る Deep Learning ❸](https://github.com/oreilly-japan/deep-learning-from-scratch-3)で題材となっているDeep Learningフレームワーク DeZeroをRubyと[Numo::NArray](https://github.com/ruby-numo/numo-narray)を使って実装してみたので、現状できることの紹介と実装時のポイントをいくつか説明します。

## DeZeroとは
オライリーから出ている書籍[ゼロから作る Deep Learning ❸](https://github.com/oreilly-japan/deep-learning-from-scratch-3)は、Step-by-StepでDeep Learningフレームワークを作るという内容となっており、そこで最終的に作られるフレームワークがDeZeroです。
DeZeroは、MLP(多層パーセプトロン)だけでなくCNNやRNNのモデルまで作ることができる本格的なフレームワークとなっています。

実装自体はPythonで書かれており、内部の重要なデータ構造である多次元配列はnumpyを使っています。
そして、このnumpyの機能があるからこそ多次元配列に対する面倒な計算処理を簡単に記述することができます。
また、numpyの計算処理自体はPythonではなくCによるネイティブコードで実行されるので非常に高速です。

以上のような特徴を持ったDeZeroをRubyで実装する場合、重要になるのがnumpyで実装されている多次元配列とその計算の部分になります。
RubyではPythonのライブラリを呼ぶことができるPyCallを使ってnumpyを使う方法があり、既にその方法でDeZeroを実装されているものがあります。
一方、Rubyにもnumpyの代替となるライブラリとして[Numo::NArray](https://github.com/ruby-numo/numo-narray)があります。

今回、そもそもDeZeroをRubyで実装しようと思ったのは、このNumo::NArrayがどれだけnumpyの代わりとして使えるのか試してみたかったというところにあります。

そこで、numpyの代わりにNumo::NArrayを使ってDeZeroの実装を始めました。

## Deep Learning Framework for Ruby

DeZeroのRuby実装ということで[Dezerb](https://github.com/koji-m/dezerb)という名前にしました。

現段階では、簡単なMLPのモデルを作れるところまで実装が完了しているので、そのデモを紹介します。

### 線型回帰の実行例

線形関数 `y = wx + b` の平均二乗誤差に対して勾配降下法を使って100エポック訓練した結果を表示します。
緑色の'x'マークで左下から右上にランダムに分布しているポイントが訓練データとなります。
紫色の直線が訓練後の線形関数で、訓練データに概ね適合しているのがわかります。

![linear regression](/images/linear_regression.png)

### ニューラルネットワークの実行例

活性化関数としてシグモイド関数を適用する２層のニューラルネットワークを10000エポック訓練した結果です。
こちらも概ね適合しています。

![neural network](/images/neural_network.png)

## 実装のポイント

## numpyとNumo::NArrayのメソッドの対応

[Numo vs numpy](https://github.com/ruby-numo/numo-narray/wiki/Numo-vs-numpy)というドキュメントに各々のメソッドの対応表があります。この表を見るとわかるのですが主要なメソッドはほとんどカバーされていることがわかります。

この対応表を見ながらnumpyのメソッドの部分を単純にNumo::NArrayのものに書き換えていけばほぼ完成です。

### Numo::NArrayに足りないメソッドをRubyで実装

配列の全要素の和を求める`sum`のような関数では、バックプロパゲーション時に配列のブロードキャストを行います。実装上、numpyではbroadcast_to関数で明示的にブロードキャストしますが、Numo::NArrayには`broadcast_to`のような明示的なブロードキャストメソッドがありません。

そこで、処理速度は犠牲になりますが`broadcast_to`メソッドをRubyで実装しました。
ブロードキャストのルール通りに、先にブロードキャスト前後の配列の次元数を揃えて(reshape)、各次元のサイズの大きい方に値を反復しながら引き伸ばす(tile)、というように[実装](https://github.com/koji-m/dezerb/blob/master/lib/dezerb/utils.rb#L5)しています。

尚、このブロードキャスト自体のバックプロパゲーションの処理にnumpyの`squeeze`を使っているのですが、こちらもNumo::NArrayには用意されていないのでRubyで[実装](https://github.com/koji-m/dezerb/blob/master/lib/dezerb/utils.rb#L36)しました。

### 演算子のオーバーロード

細かいことですが、PythonとRubyで演算子をオーバーロードしたときの挙動が異なるため、その実装の違いを取り挙げておきます。

計算式の変数を表すクラスとして`Variable`クラスを実装しています。この`Variable`オブジェクト同士の計算だけでなく`Variable`オブジェクトとNumo::NArrayオブジェクトの計算をシームレスに記述できるように演算子をオーバーロードしています。

Pythonの場合、演算子のオーバーロードは`__演算子名__`(ex. `__mul__`)メソッドと`__r演算子名__`メソッドを定義することで、Pythonのビルトイン数値オブジェクトと`Variable`オブジェクトの演算をシームレスに記述できます。また、ndarrayと`Variable`オブジェクトの演算も記述できるように`Variable`クラスのインスタンス属性`__array_priority__`(演算子の優先度)の値をndarrayより高い値に設定しています。

Rubyでも同じことを実現するために、`Variable`クラスに`coerce`メソッドを定義し、その中で非`Variable`オブジェクトの項を`Variable`オブジェクトに変換(キャスト)しています。

## 最後に

DeZeroをRubyで再実装することで、ニューラルネットワークの計算手法について改めて学び直すことができました。また、具体的な実装をすることで理解があやふやであったところが明確になったと感じています。更には、実装の過程で、今まで使ったことのなかったRubyの言語機能を知ることができるなど副次的な学びもありました。

そして主目的であるNumo::NArrayがnumpyの代わりとしてどれだけ使えるのかという点については、今回の利用の範囲ではほぼ完璧に代替として使えることがわかりました。(broadcast_toやsqueezeもそのうち実装されることを願っています)

今後は引き続き、CNNやRNNのモデルが作れるところまで実装を進めていこうと思います。また、GPU対応として[Cumo](https://github.com/sonots/cumo)を取り入れてみたり、GBDT(勾配ブースティング決定木)などニューラルネットワーク以外のモデルも作れるようにすると面白そうです。
