# はじめに
\label{sec:introduction}

Elixir\cite{Elixir}は，並行・並列プログラミングに長けており，耐障害性が高いという特長を備えている．さらに， Elixir で書かれたウェブサーバーフレームワークである Phoenix\cite{Phoenix}を用いることで，極めてレスポンス性の高いウェブサーバーを構築できる\cite{Elixir16}．Elixir User's Survey 2016 \cite{ElixirSurvey2016} の調査対象で Elixir を採用していると回答した企業数は，全世界で2014年に191，2015年に405，2016年に1109である．現在はさらに急速に普及が進んでいる．本論文では，第\ref{sec:Elixir}章にて Elixir の特長を詳述する．

Elixir の高い並行・並列プログラミング能力と耐障害性を支えているのは，Elixir の実行系である Erlang VM である．Erlang VM は Erlang \cite{Erlang} のために開発された VM で，Elixir の他にもいくつかのプログラミング言語が Erlang VM 上で動作する．2018年になって，Erlang VM と互換性をもつ実行系が相次いで提案・発表されている\cite{AtomVM}\cite{CoreErlang}\cite{Starlight}．本論文では，第\ref{sec:ErlangVM}章にて Erlang VM の特長を詳述し，互換実行系について概説する．

我々が本発表で提案する Elixir のサブセット言語であるプログラミング言語 micro Elixir とその処理系 ZEAM (ZACKY's Elixir Abstract Machine) では，そのような Erlang VM 互換のアプローチとは異なるアプローチで Elixir の処理系を開発する．

おそらく新たな処理系をわざわざ研究開発することについて，次のような疑問を持つことだろう:

* 現在主流の Erlang VM から円滑に移行することはできるのか？
* 優れた Erlang VM よりもさらに優れた処理系を作れる勝算はあるのか？
* すでにたくさんの Erlang VM 互換のプログラミング言語処理系が数多く提案されている中で，さらに micro Elixir / ZEAM を研究開発していくことに意義はあるのか？

第\ref{sec:implementationStrategy}章でこれらの問いに対する答えを説明する．

micro Elixir / ZEAM は次のような構成である:

* Elixir マクロを用いたメタプログラミング解析器 (第\ref{sec:analyzer}章)
* LLVM を用いたコード生成系 (第\ref{sec:generator}章)

第\ref{sec:parallelismCategories}章に我々が考える Elixir の並列性の3分類を提案する．その上で，我々が micro Elixir / ZEAM で掲げる野心的な研究目標は次の通りである:

* 命令並列性に基づく静的命令スケジューリング (第\ref{sec:instructionScheduling}章)
* CPU / GPU を統合する超並列高速実行処理系 Hastega (第\ref{sec:Hastega}章)
* I/Oバウンド処理を高速化する省メモリ並行プログラミング機構 Sabotender (第\ref{sec:Sabotender}章)
* 実行時間予測に基づく静的タスクスケジューリングとハードリアルタイム性 (第\ref{sec:executionTimeEstimation}章)
* プロセス間通信を含む超インライン展開(第\ref{sec:superInlining}章)
* 超インライン展開や静的タスクスケジューリングを前提にした大域的なキャッシュメモリとI/Oの最適化(第\ref{sec:globalOptimization}章)

本論文の目的は micro Elixir / ZEAM の研究構想を提示し，各研究目標に対する技術的課題を明らかにすることである．

これらの研究目標の実現を前提に，今までの常識を打ち破るようなカーネルやVM，プロセッサアーキテクチャ，並列/分散コンピューティング，特定ドメイン向け最適化などの研究に取り組む．本論文のまとめを第\ref{sec:summary}章に，将来課題を第\ref{sec:futureWorks}章にそれぞれ述べる．


# Elixir
\label{sec:Elixir}

Elixir は MapReduce プログラミングスタイル\cite{Dean:2008:MSD:1327452.1327492}を採用している．このような Elixir のプログラム例を図\ref{fig:mapreduce-elixir-code}に示す．

\begin{figure}[h]

  \begin{center}
    \begin{BVerbatim}
1..1_000_000
|> Enum.map(foo)
|> Enum.map(bar)
|> IO.inspect
    \end{BVerbatim}
  \end{center}
  \caption{MapReduce プログラミングスタイルの Elixir コード例}\label{fig:mapreduce-elixir-code}
\end{figure}

まず，1行目の `1..1_000_000` は，1から1,000,000までの要素からなるリストを生成する．なお，数字の間の `_` によって，数字を分割するコンマを表す．

次に，2,3,4 行目の先頭にある `|>` はパイプライン演算子である．パイプライン演算子の前に書かれている記述の値を，パイプライン演算子の後に書かれた関数の第1引数として渡す．すなわち，次のような記述と等価である．

```
IO.inspect(Enum.map(Enum.map(1..1_000_000, foo), bar))
```

パイプライン演算子により，左から右へ，上から下へ，自然な流れで処理を記述することができ，Lisp にように括弧の複雑なネストにより可読性を損ねることがなくなる．

次に 2,3 行目に書かれている `Enum.map` は，第1引数に渡されるリスト等の要素1つ1つに，第2引数で渡される関数を適用する．ここでは，関数 `foo` を各要素に適用した後，関数 `bar` を各要素に適用する．
最後に，4行目の `IO.inspect` は第1引数に渡された値の内部表現を表示する．

もし，関数 `foo` が元の値を2倍する関数で，関数 `bar` が元の値に1加える関数であった場合には，図\ref{fig:mapreduce-elixir-code}のプログラムにより，2倍してから1加える処理を1から1,000,000までの要素に適用したリスト，すなわち `[3, 5, 7, ...]` を生成して表示する．


# Erlang VM
\label{sec:ErlangVM}

現在，Elixir は Erlang VM という仮想機械(VM)上で動作する．Erlang VM の特長は，並行プログラミングに優れていることと，耐障害性が高いことである．

並行プログラミングに優れる理由は，アクターモデル\cite{Hewitt:1973:UMA:1624775.1624804}に基づく並行プログラミングモデルを採用していること，プロセスの生成がコアごとに独立していて軽量であること，資源を直接操作するプロセスを1つに限定して資源の利用をそのプロセスへのメッセージパッシングで実現していること，これらにより同期・排他制御を行う必要性を大幅に削減していることが挙げられる．

また，耐障害性に優れている理由は，プロセスごとに分離したメモリ空間であること，そのことにより Full GC が働いていわゆる Stop the world すなわち処理系を利用するすべてのプログラムの動作が一斉停止する事態に陥ることがないこと，基本的にプログラム中に例外処理を記述せずにプロセスごと異常終了するように設計しておき監視プロセスにより該当プロセスを再起動させることで復旧させるアプローチを採用していること，これらのことにより障害が発生してもメモリのリークや不整合が生じることがないことなどが挙げられる．

Elixir は，このような Erlang VM の特長を継承している．

(他の Erlang VM 互換の概説)

# micro Elixir / ZEAM の実装戦略
\label{sec:implementationStrategy}

第\ref{sec:introduction}章での問いを再掲する:

* 現在主流の Erlang VM から円滑に移行することはできるのか？
* 優れた Erlang VM よりもさらに優れた処理系を作れる勝算はあるのか？
* すでにたくさんの Erlang VM 互換のプログラミング言語処理系が数多く提案されている中で，さらに micro Elixir / ZEAM を研究開発していくことに意義はあるのか？

以下に各問いについて答えていく．

## Erlang VM からの円滑な移行戦略
\label{sec:smoothStrategy}

まずは「現在主流の Erlang VM から円滑に移行することはできるのか？」という問いに答える．

我々は Erlang VM 互換の処理系を一から研究開発するアプローチの問題点を次のように考えた:

* 現行の優れた言語処理系である Erlang VM と両立しない
* そのことにより，少なくとも Erlang VM 同等以上のパフォーマンスを安定して得られるようにならないと実運用に用いる利点がない
* そのような状態になるまでに，多大な時間を必要とする

そこで，我々が採用した戦略は次の通りである:

1. Elixir のサブセットのプログラミング言語である micro Elixir を新たに定義する
2. ZEAM は Elixir プロジェクト中のコードの一部を，NIF (Native Implemented Function: ネイティブコードで実装された関数) にコンパイルして Erlang VM から呼び出せるようにする処理系として，当面の間研究開発を進める
3. \label{enum:splitCode} ZEAM は，与えられたコードを解析して，micro Elixir の言語仕様の範囲内のコードと範囲外のコードに分割する．前者はネイティブコードにコンパイルして NIF として定義する．NIF呼出しコードと後者のコードを繋ぎ合わせた Elixir コードを生成し，Erlang VM で実行するようにする
4. ZEAM の解析部は， Elixir マクロを利用することで，パーサーをフルスクラッチで開発しないで済ませる (第\ref{sec:analyzer}章)
5. ZEAM の生成部は， Rust \cite{Rust} 経由で LLVM \cite{LLVM} を利用することで，対応可能な ISA を最大化し，かつ最適化器などのツールチェーンを活用できるようにする   
6. \label{enum:Hastega} ZEAM の最初のアプリケーションを超並列高速実行処理系 Hastega (第\ref{sec:Hastega}章)とすることで，最初から micro Elixir / ZEAM を利用する動機を作る

このうち戦略\ref{enum:splitCode}について説明する．例えば図\ref{fig:mapreduce-elixir-code}のコードのうち，1〜3行目は micro Elixir の範囲内で，4行目の `IO.inspect` は範囲外であるとする(最初期にはこのようにデザインする予定である)．この時，図\ref{fig:splitCode}のように分割し，関数 `compile_to_nif` をネイティブコードにコンパイルして NIF として定義する．なお，`def func do ... end` は引数のない関数 `func` を `...` で示されるコードを実行する関数として定義することを意味する Elixir の構文である．

\begin{figure}[h]

  \begin{center}
    \begin{BVerbatim}
def compile_to_nif do
  1..1_000_000
  |> Enum.map(foo)
  |> Enum.map(bar)
end

def rest_elixir_code do
  compile_to_nif
  |> IO.inspect
end
    \end{BVerbatim}
  \end{center}
  \caption{コード分割例}\label{fig:splitCode}
\end{figure}

すなわち，micro Elixir / ZEAM は Erlang VM 互換を目指すのではなく，Elixir に特化してより高度に最適化したプログラミング言語処理系を目指す戦略を採っている．Erlang VM 互換を捨てる代わりに，最初のうちは Erlang VM から呼出して利用しやすい仕組みとして研究開発を進めることで，Erlang VM 互換戦略よりも早期に実務で利用できるようにした．

## Erlang VM を超えていくロードマップ
\label{sec:beyondErlangVM}

次に「優れた Erlang VM よりもさらに優れた処理系を作れる勝算はあるのか？」という問いに答える．

\ref{sec:smoothStrategy}節で述べた戦略\ref{enum:Hastega}で示した Hastega (第\ref{sec:Hastega}章) は，現行の Erlang VM には備わっていない機能である．しかも，Erlang VM に機能追加する形で実現することので，実現した暁には，Elixir / Phoenix から Hastega の機能を自由に利用できるようになる予定である．この時点で，Erlang VM より優れた処理系を作っていることになると考えている．

さらに第\ref{sec:introduction}章でも提示したように次のような野心的な研究目標を掲げている:

* 命令並列性に基づく静的命令スケジューリング (第\ref{sec:instructionScheduling}章)
* I/Oバウンド処理を高速化する省メモリ並行プログラミング機構 Sabotender (第\ref{sec:Sabotender}章)
* 実行時間予測に基づく静的タスクスケジューリングとハードリアルタイム性 (第\ref{sec:executionTimeEstimation}章)
* プロセス間通信を含む超インライン展開(第\ref{sec:superInlining}章)
* 超インライン展開や静的タスクスケジューリングを前提にした大域的なキャッシュメモリとI/Oの最適化(第\ref{sec:globalOptimization}章)

これらの研究目標の中には，NIF のような形で外付けしたり，現行の Erlang VM を修正して機能を取り入れることが難しいものも存在する．例えば，Sabotender (第\ref{sec:Sabotender}章)の実装には，プロセスとI/Oアクセスの実装に相当手を入れる必要がある見込みであり，Erlang VM に機能追加するのは極めて困難であると考えられる．

そこで，micro Elixir / ZEAM の最終形の1つとして，Erlang VM などの他の処理系に依存することのない，独立した処理系を提供することを考えている．実装が進み，micro Elixir の言語仕様の範囲を十分に広げて Elixir の処理系として実用上使える状態にまでなれば可能になるだろう．その主な目的の1つは，Erlang VM では実現できないような研究目標を達成することである．

## 他の Erlang VM 互換のプログラミング言語処理系との関係性

次に「すでにたくさんの Erlang VM 互換のプログラミング言語処理系が数多く提案されている中で，さらに micro Elixir / ZEAM を研究開発していくことに意義はあるのか？」という問いに答える．

\ref{sec:beyondErlangVM}節に述べたように，当面は既存の Erlang VM に機能付加する形で micro Elixir / ZEAM の研究開発を進める．この戦略は他の Erlang VM 互換のプログラミング言語処理系についても同様に考えることができる．すなわち，これらの研究開発者と協力関係が築けるのであれば，それらの処理系についても，技術的に可能な限り，Hastega などの micro Elixir / ZEAM の機能を利用できるようにしたいと考えている．


# Elixir マクロを用いたメタプログラミング解析器
\label{sec:analyzer}

Elixir には Elixir マクロという強力なメタプログラミング機構が備わっている．Elixir マクロを用いることで，Elixir の言語仕様を容易に拡張できるだけでなく，既存の言語仕様のパースを記述することなく Elixir プログラム中で Elixir プログラムの 抽象構文木 (Abstract Syntax Tree: AST) を参照・操作できる．

例えば図\ref{fig:mapreduce-elixir-code}のコードは，Elixir マクロにより下記のような AST に変換される:

```
{:|>, [context: Elixir, import: Kernel],
 [
   {:|>, [context: Elixir, import: Kernel],
    [
      {:|>, [context: Elixir, import: Kernel],
       [
         {:.., [context: Elixir, import: Kernel], [1, 1000000]},
         {{:., [], [{:__aliases__, [alias: false], [:Enum]}, :map]}, [],
          [{:foo, [], Elixir}]}
       ]}, 
      {{:., [], [{:__aliases__, [alias: false], [:Enum]}, :map]}, [],
       [{:bar, [], Elixir}]}
    ]},
   {{:., [], [{:__aliases__, [alias: false], [:IO]}, :inspect]}, [], []}
 ]}
```

この AST の詳説は割愛するが，この AST は Elixir の基本データ構造であるアトム `:atom` とタプル `{a, b, c, ...}` ，リスト `[a, b, c, ...]` を組合わせて表現されているので，通常の Elixir のプログラムにより変換や解析を行うことができる．

micro Elixir / ZEAM では Elixir マクロを解析部に用いることで，通常のプログラミング言語処理系の実装で必要になるパースを記述する必要性が無くなり，AST を解析して中間コードを生成する本質的なプログラミングに集中できるようにした．

# LLVM を用いたコード生成系
\label{sec:generator}

近年のコンパイラではコード生成系として LLVM \cite{LLVM} が採用されていることが多い．LLVM の利点は，実に多様なアーキテクチャのコードを生成できることと，豊富な最適化器のツールチェーンを活用できることである．

したがって，我々も micro Elixir / ZEAM のコード生成系として LLVM を採用したい．しかし，2019年1月現在，Elixir から LLVM を利用するためのバインディングと呼ばれる API はまだ提供されていない．

そこで，我々は Elixir から Rust \cite{Rust} を呼出し，Rust の LLVM バインディングを利用することで，Elixir から LLVM を用いてコード生成できるようにする．Elixir と Rust のインタフェースは Rustler \cite{Rustler} を用いる．

# Elixir コードの並列性の3分類
\label{sec:parallelismCategories}

Elixir コードから読取れる並列性には，大きく分けて次の3種類があると我々は考えている:

* 命令並列性: Elixir コードをデータフローで表したときに現れる並列性
* Hastega 型並列性: 例えば図\ref{fig:mapreduce-elixir-code}のコードで示されるような MapReduce プログラミングスタイルによって示唆される並列性
* Sabotender 型並列性: アクターモデル\cite{Hewitt:1973:UMA:1624775.1624804}にしたがった並行プロセスモデルで現れる並列性

それぞれの並列性を考慮して，次のように並列化・高速化を行う:

* 命令並列性: 第\ref{sec:instructionScheduling}章
* Hastega 型並列性: 第\ref{sec:Hastega}章
* Sabotender 型並列性: 第\ref{sec:Sabotender}章

# 命令並列性に基づく静的命令スケジューリング
\label{sec:instructionScheduling}

# CPU / GPU を統合する超並列高速実行処理系 Hastega
\label{sec:Hastega}

図\ref{fig:mapreduce-elixir-code}のプログラムコードは容易に並列実行が可能である．1から1,000,000の各要素に対する関数 `foo` と関数 `bar` の適用は他に影響されることなく完全に独立して実行することが可能である，すなわちこのプログラムは 1,000,000 の並列性を持つことが容易に推測できる．Flow \cite{Flow} という並列処理ライブラリは，このことを利用してマルチコアCPUによる並列プログラミングを簡便にする方法を実現した．また，我々も Hastega という CPU / GPU を統合する超並列高速実行処理系を提案し，研究開発している\cite{ZACKY18J}\cite{ZACKY19-Hastega}．


# I/O バウンド処理を高速化する省メモリ並行プログラミング機構 Sabotender
\label{sec:Sabotender}


# 実行時間予測に基づく静的タスクスケジューリングとハードリアルタイム性
\label{sec:executionTimeEstimation}

# プロセス間通信を含む超インライン展開
\label{sec:superInlining}

# 超インライン展開や静的タスクスケジューリングを前提にした大域的なキャッシュメモリと I/O の最適化
\label{sec:globalOptimization}

# まとめ
\label{sec:summary}


# 将来課題
\label{sec:futureWorks}

