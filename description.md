# はじめに

Elixir\cite{Elixir}は，並行・並列プログラミングに長けており，耐障害性が高いという特長を備えている．さらに， Elixir で書かれたウェブサーバーフレームワークである Phoenix\cite{Phoenix}を用いることで，極めてレスポンス性の高いウェブサーバーを構築できる\cite{Elixir16}．Elixir User's Survey 2016 \cite{ElixirSurvey2016} の調査対象で Elixir を採用していると回答した企業数は，全世界で2014年に191，2015年に405，2016年に1109である．現在はさらに急速に普及が進んでいる．本論文では，第\ref{sec:Elixir}章にて Elixir の特長を詳述する．

Elixir の高い並行・並列プログラミング能力と耐障害性を支えているのは，Elixir の実行系である Erlang VM である．Erlang VM は Erlang \cite{Erlang} のために開発された VM で，Elixir の他にもいくつかのプログラミング言語が Erlang VM 上で動作する．2018年になって，Erlang VM と互換性をもつ実行系が相次いで提案・発表されている\cite{AtomVM}\cite{CoreErlang}\cite{Starlight}．本論文では，第\ref{sec:ErlangVM}章にて Erlang VM の特長を詳述し，互換実行系について概説する．

我々が本発表で提案する Elixir のサブセット言語であるプログラミング言語 micro Elixir とその処理系 ZEAM では，そのような Erlang VM 互換のアプローチとは異なるアプローチで Elixir の処理系を開発する．

おそらく新たな処理系をわざわざ研究開発することについて，次のような疑問を持つことだろう:

* 現在主流の Erlang VM から円滑に移行することはできるのか？
* 優れた Erlang VM よりもさらに優れた処理系を作れる勝算はあるのか？
* すでにたくさんの Erlang VM 互換のプログラミング言語処理系が数多く提案されている中で，さらに micro Elixir / ZEAM を研究開発していくことに意義はあるのか？

第\ref{sec:implementationStrategy}章でこれらの問いに対する答えを説明する．

micro Elixir / ZEAM は次のような構成である:

* Elixir マクロを用いたメタプログラミング解析器 (第\ref{sec:analyzer}章)
* LLVM を用いたコード生成系 (第\ref{sec:generator}章)

我々が micro Elixir / ZEAM で掲げる野心的な研究目標は次の通りである:

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

# micro Elixir / ZEAM の実装戦略
\label{sec:implementationStrategy}


# Elixir マクロを用いたメタプログラミング解析器
\label{sec:analyzer}

# LLVM を用いたコード生成系
\label{sec:generator}

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

