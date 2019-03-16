# はじめに

Cisco \cite{Cisco2018}によると，インターネット全体のIPトラフィックは，2017年に毎月122エクサバイト増加している．この増加率は年々増大しており，2022年には毎月396エクサバイト増加すると見込んでいる．また，IDC \cite{IDC2014}によると，インターネット上のデータの総量は2013年に4.4ゼタバイトであるが，2020年には44ゼタバイトに増加すると見込まれている．これらが示すように世界に流通する情報量は急速に増大している．

この問題に対処するためには，情報を処理する計算能力を情報量の流通量と同等以上に高める必要がある．しかし，Hennesy と Patterson \cite{HenessyPatterson2017}によると，2003年までは年々順調にクロック周波数が増加していたのに対し，2003年ごろから頭打ちとなってしまっている．これはクロック周波数が増大すると消費電力と発熱量も増大するが，電源供給量と熱伝導率および冷却能力が追いつかなくなり，常温の環境下で安定して動作させることが不可能になってしまうためである．

2003年以降のプロセッサの進化はクロック周波数の増加ではなくコア数の増加によっている．Intel が2006年に販売した Core シリーズの最上位プロセッサである Intel Core 2 Extreme X6800 は，クロック周波数が2.93GHz，物理コア数は2，論理コア数は4である．一方，2017年に販売した Core シリーズの最上位プロセッサである Intel Core i9 7980XE は，クロック周波数が2.6GHz，物理コア数は18，論理コア数は36である．

クロック数が順調に伸びていた時代ではソフトウェアを何も変えなくても順調に性能が向上していたであろう．一方，クロック周波数が伸びずにコア数が増加する状況下で性能を向上させるためには，並列プログラミングが求められる．そこで，近年次々と，さまざまな並列プログラミングモデルに基づいたプログラミング言語が提案されている．

Elixir (エリクサー) \cite{Elixir}は2012年に Jos\'{e} Valim が開発した並列プログラミング言語である．Elixir は関数型言語 Erlang \cite{Erlang}を母体としており，並列プログラミングのための数々の優れた特長を有する．

Elixir を用いることで，たとえばウェブシステムのレスポンス性能を大きく改善することができる．Fedrecheski ら\cite{Elixir16}によると，Java では毎秒1,200リクエスト程度で急速にレスポンス性能が悪化してしまったのに対し，Elixir では毎秒1,800リクエスト程度まで耐えられる．

このような背景で我々は Hastega (ヘイスガ)のプロトタイプを開発した\cite{ZACKY18J, Hisae19J, ZACKY19-Hastega, ZACKY19-Hastega-Medium, ZEAM-log}．Hastega は，ウェブアプリケーションの中で行う画像処理や機械学習などの計算負荷の高い処理を，Elixirで簡潔に記述し，サーバー上やエッジ，ウェブクライアント上の CPU や GPU に負荷分散させながら高速に並列実行させることを目指している\cite{ZACKY19-Hastega, ZACKY19-Hastega-Medium}．Hastega の実現により，計算量爆発の問題に対抗するための有効な計算能力を容易に向上できる．

整数演算を行うベンチマークでは，Hastegaのプロトタイプは，元のElixirコードの4-8倍の速度向上，Python \cite{Python}のCuPy\cite{CuPy}を用いてGPUを駆動したコードの3倍以上の速度向上を得られた\cite{ZACKY18J}．線形回帰を行うベンチマークでは，Hastegaのプロトタイプは，元の Elixir コードの5-17倍，インライン展開した Elixir コードの4-8倍の速度向上が得られた\cite{Hisae19J}．この線形回帰のベンチマークでは，データの流量が増大すると並列化のためのコストの元が取れて速度向上比が大きくなる様子が観測されたことと，データ転送に時間がかかることと，分配アルゴリズムがコアに線形に展開するアルゴリズムであることから，分配するコア数が増えると極端に性能が悪化する様子が観測された\cite{Hisae19J}．

Hastega を本格的に実装するにあたり，(1) Elixirで書かれたコード中のパイプライン演算子とmap関数を用いた記述部分を抽出する部分と，(2) 抽出したコードを並列化・最適化したネイティブコードにコンパイラする部分の，2つの構成要素に分割する設計方針を採用した．本報告で提案・報告する SumMag (サムマグ)は，前者(1)を Elixir マクロを用いて実装したメタプログラミングライブラリである．

Elixir は，リストとタプルからなる抽象構文木(AST)を操作するようなメタプログラミング機構，Elixir
マクロを提供している．そこで我々は，SumMag の実装にElixir マクロを用いた．

原理としてはパイプライン化された1つ以上の map 関数の適用をまとめて抽出して関数として分離し，元の Elixir コードの該当部分を抽出した関数への呼び出しに置き換える．このときに
map関数で適用される関数を Elixir コードのレベルでインライン展開し，かつ
map-map フュージョンを行う．

本報告のこの後の構成は次のとおりである: 第\ref{sec:Elixir}章では，Elixir について他言語との関連や名称の由来などについて紹介する．第\ref{sec:Zen}章では，3つのプログラミングスタイルについて触れ，そのうちの1つを特に重要視していることを詳述する． 第\ref{sec:map-map}章では，map関数の最適化であるmap-mapフュージョンを紹介し，第\ref{sec:Hastega}章で，Hastega について，原理や構成要素，名称の由来などについて詳述し，原理のところで map-map フュージョンの必要性についても説明する． 第\ref{sec:macro}章では，Elixir のメタプログラミング機構である Elixir マクロについて紹介し，例示する．第\ref{sec:design}章で 第\ref{sec:summary}章で 

# Elixir (エリクサー)
\label{sec:Elixir}

Elixir(エリクサー) \cite{Elixir}は2012年に Jos\'{e} Valim が開発した並列プログラミング言語である．Jos\'{e} は  Ruby on Rails \cite{RoR}のコミッタの1人でもあることから，Elixir は Ruby \cite{Ruby} の影響を強く受けている．Elixir開発当時では，Ruby には並列プログラミング機構が十分備わっていなかったことから，アクターモデル\cite{Hewitt:1973:UMA:1624775.1624804}に基づく並行プログラミングモデルを採用するなど，並列プログラミングのための数々の優れた特長を有する関数型言語 Erlang \cite{Erlang}を母体とした．

このような背景から，Elixir は Ruby on Rails と同等のウェブアプリケーションフレームワークを実現するのに必要な機能を盛り込んだ言語仕様を備えており，その1つがメタプログラミング機構である．Elixir のメタプログラミング機構は，LISP \cite{McCarthy:1960:RFS:367177.367199}のメタプログラミング機構と似ており，タプルとリストで構成される抽象構文木 (AST)を操作することで行う．プログラムコードから AST を生成する関数が `quote` であり，AST に新しいコードや値を挿入する関数が `unquote` である．この詳細については，第\ref{sec:macro}章で述べる．Elixir は LISP 同様，核となる言語仕様(Elixir カーネル言語)をメタプログラミングを用いて拡張している．Elixir は，Elixir カーネル言語を，Erlang VM のバイトコード BEAM にコンパイルすることで， Erlang VM で実行できるようにしている．

以上のようなことから，「Elixir は，Erlang を母とし，Ruby を父とし，LISP を祖父に持つプログラミング言語である」と俗に言われる．

Ruby における Ruby on Rails に相当する Elixir のウェブアプリケーションフレームワークが，Phoenix \cite{Phoenix} である．Phoenix はレスポンス性能が極めて高いことから，オンラインゲームのバックエンドとして用いられる採用事例が目立つ．Elixir と Phoenix は後発であるにも関わらず，普及が目覚ましく，2016年の時点で1109の企業がプロダクション採用している\cite{ElixirUsersSurvey}．

Elixir と Phoenix の名は，ファイナルファンタジーに登場するアイテムと幻獣の名称に由来する．ファイナルファンタジーから名をとったのは，これらの創始者である Jos\'{e} Valim と Chris McCord をはじめとするコミッターたちがファイナルファンタジーオンラインの大ファンであったからである．Elixir はファイナルファンタジーでは全回復する薬として描かれているが，もともとは錬金術師が求めた不老不死の薬の名称である．一方 Phoenix は，自分の身を炎の中に投じて再び蘇る伝説上の不老不死の炎の鳥の名称である．このように Elixir と Phoenix が不老不死の象徴の名称を取るのには理由がある．それは Elixir と Phoenix が耐障害性が極めて高いからである．Elixir は，並行プログラミング機構として，メモリ空間を GC を含めて完全に分離するプロセスモデルを採用している．また，監視プロセスを用いて，異常終了したプロセスを検知して再起動する仕組みを備えている．Phoenix ではこれらを利用して，高負荷をかけたり不正なデータを送信したりしても，異常終了したプロセスを監視プロセスによって再起動させる設計にしている．このような様子が不老不死であるように見えることから，Elixir と Phoenix という不老不死の象徴の名称をつけている．

Elixir のコード例については，次の第\ref{sec:Zen}章で紹介する．

# Elixir Zen Style

\label{sec:Zen}

\begin{figure*}[t]
\centering
  \begin{tabular}{c}

    \begin{minipage}{0.5\hsize}
    \centering
{\small
\begin{verbatim}
1..1_000_000
|> Enum.to_list
|> R.func()

defmodule R do
	def func( [] ), do: []
	def func( [ head | tail ] ) do
		[ head |> M.foo() |> M.bar() 
		| func(tail) ]
	end
end

defmodule M
	def foo(n), do: n * 2
	def bar(n), do: n + 1
end
\end{verbatim}
}
		\captionof{figure}{再帰呼出スタイルの Elixir コード例}
		\ecaption{An Elixir code example using recursive call}
		\label{fig:recursive}
		\end{minipage}

    \begin{minipage}{0.5\hsize}
    \centering
{\small
\begin{verbatim}
1..1_000_000
|> Enum.map(&M.foo(&1))
|> Enum.map(&M.bar(&1))

defmodule M
	def foo(n), do: n * 2
	def bar(n), do: n + 1
end
\end{verbatim}
}
		\captionof{figure}{Enum.map スタイルの Elixir コード例 (Elixir Zen Style)}
		\ecaption{An Elixir code example using Enum.map}
		\label{fig:zen}
		\end{minipage}
	\end{tabular}
\end{figure*}


\begin{figure}[b]
\centering
{\small
\begin{verbatim}
int i;
int [] array = new int[1000000];
for(i = 0; i < 1000000; i++)
	array[i] = i + 1;
for(i = 0; i < 1000000; i++)
	array[i] = foo(array[i]);
for(i = 0; i < 1000000; i++)
	array[i] = bar(array[i]);
\end{verbatim}
}
\caption{ループスタイルの Java コード例}
\ecaption{A Java code example using loop}
\label{fig:loop}
\end{figure}


\figref{fig:recursive}は，関数型言語で一般的な再帰呼出スタイルによるElixirプログラム例である．コードの説明は次のとおりである:

* `1..1_000_000` は，1から1,000,000までの整数の範囲を表す．
* `Enum.to_list` は，引数で与えられた値をリストに変換する．
* `|>` は**パイプライン演算子**である．パイプライン演算子は2項演算子であり，左辺の値を右辺の関数の第1引数として与え，右辺の関数の引数を第2引数以下に繰り下げて，関数を呼び出す．
* したがって，`1..1_000_000 |> Enum.to_list` は，`Enum.to_list(1..1_000_000)`と等価である．すなわち，1から1,000,000までの整数を要素として持つリストを生成する．
* 同様に `1..1_000_000 |> Enum.to_list |> R.func()` は，`R.func(Enum.to_list(1..1_000_000))` と等価である．パイプライン演算子により，コードを読むときに，左から右へ，上から下へ，自然な流れで読むことができるようになり，括弧のネスティングにより可読性が落ちることを防ぐ．
* `R.func` は，モジュール`R`に定義されている関数`func`のことである．
* `defmodule M do ... end` は，`...` を本体として持つモジュール`M`を定義する．
	* `def func(arg) do ... end` により，`...` を本体として持つ，引数 `arg` の関数 `func` を定義する．`do ... end` が1行で記述できる場合には，`def func(arg), do: ...` と書いても良い．
	* 引数の数をアリティと呼ぶ．異なるアリティを持つ同名の関数は，異なる関数として区別される．Elixirでは関数を厳密に区別するために，たとえば`R.func/1`というような記法を用いる．これは，モジュール`R`で関数名`func`，アリティが1の関数，という意味である．
	* 関数`M.foo/1`は，第1引数の値を2倍した値を返す関数である．同様に関数`M.bar/1`は，第1引数の値を1加えた値を返す関数である．このような関数はLisp同様に無名関数で定義することもできるが，本報告では説明を割愛した．
* 同じアリティを持つ同名の関数の定義が複数あるときには，なんらかの条件で分岐して実行される．たとえば，関数`R.func/1`では，空リスト`[]`を引数にする場合が先に定義されていて，リスト `[head | tail]` が引数である場合が後で定義されている．この場合，先に引数が空リストにマッチするかを評価し，マッチすれば空リストの場合の関数定義を実行する．マッチしなければ，次の定義であるリストにマッチするかを評価し，マッチすればリストの場合の関数定義を実行する．マッチしなければ，`FunctionClauseError`を発生させる．このように動的に呼び分ける仕組みを，**関数パターンマッチ**と呼ぶ．
* 関数`R.func/1`の引数が空リスト`[]`である場合には，空リストを返す．
* 関数`R.func/1`の引数が空リストではないリストである場合には，LISPの`car`にあたる部分を`head`に，`cdr`にあたる部分を`tail`にそれぞれ束縛する．返す値は，`head`に`M.foo/1`と`M.bar/1`を適用して得られた値を先頭要素とし，`R.func/1`に`tail`を与えて再帰的に呼び出して得られた値を末尾要素としたものを返す．
* これらにより全体として，1から1,000,000までの要素からなるリストの各要素に対し，関数`M.foo/1`と`M.bar/1`を適用して得られる値からなるリストを生成する．

これと同じ結果が得られる別の書き方のコードを\figref{fig:zen}に示す．このコードの説明は次のとおりである:

* `Enum.map/2`は，第1引数をリストとして扱い，その各要素に対し，第2引数で与えられる関数呼出を行う．
* `&M.foo(&1)` は，無名関数定義の簡易記法で表しており，意味としては`&1`は第1引数を表しており，`M.foo/1`にそのまま第1引数を渡して呼び出しをする．
* `Enum.map/2`で抽出された各要素が第1引数として与えられるので，この値を当てはめて`M.foo/1`を呼び出すことになる．
* したがってこれらにより，`1..1_000_000 |> Enum.map(&M.foo(&1)) |> Enum.map(&M.bar(&1))`により，1から1,000,000までの各要素に対し `M.foo` と `M.bar` を適用した値を要素とするリストを返す．

参考までにループを使って書いた場合を\figref{fig:loop}に示す．Elixirではループを使っては記述できないので，Javaで書いた．

我々はLonestar ElixirConf 2019にて，\figref{fig:zen}のスタイルが最もシンプルで，読みやすく保守性に優れることを指摘し，このスタイルを**Elixir Zen Style**と呼ぶことにしよう，と提案した\cite{ZACKY19-Hastega}．\figref{fig:zen}のスタイルが禅であると我々が考えた理由は次のとおりである:

* 禅とは「本質美」である．
* `Enum.map` を使ったプログラミングスタイル (\figref{fig:zen})は，データ変換のフローを記述している．
* プログラミングをデータ変換のフローとして捉えるパラダイムは，プログラミングの本質の1つを捉えている，単純だが表現性豊かな考え方だと我々は考える．
* そこで， `Enum.map` を使ったプログラミングスタイル(\figref{fig:zen})は，本質美である，すなわち禅であると我々は考える．

これに対し，ループスタイル (\figref{fig:loop}) が禅でないと我々が考えたのは，ループカウンターという余計な存在がないと配列を走査するループを表せないこと，破壊的更新を伴うことが，本質から遠のく要因となっているからである．これらの要因によって，並列化を妨げることになるというのは特筆すべきことだろう．

再帰スタイル (\figref{fig:recursive}) が禅であるかどうかについては，もしかすると意見が分かれるところかもしれない．我々は禅でないと捉えている．Elixirプログラミングにおいては，Elixir Zen Style のようにデータ変換のフローのみで記述する方が，再帰スタイルよりも可読性と保守性に優れると我々は考えている．

この価値観があるため，我々は Elixir を関数型パラダイムとして捉えるよりも，別のパラダイムとして捉えるべきなのではないかと考えている．たとえば**データ変換型パラダイム**とでも呼ぶべきなのではないだろうか．

# map-map フュージョン
\label{sec:map-map}

map-map フュージョンは，関数型言語において一般的によく用いられるmap関数に関する最適化技術の1つで，複数の map 呼出しを1つの map 呼出しにまとめることで高速化する．\figref{fig:map-map}は，\figref{fig:zen}のコードに map-map フュージョンを適用した例である．

\begin{figure}[t]
  \centering
{\small
\begin{verbatim}
1..1_000_000
|> Enum.map( &1 |> M.foo() |> M.bar() )
\end{verbatim}
}
	\caption{図\ref{fig:zen}のコードにmap-mapフュージョンを適用した例}
	\ecaption{An application of map-map fusion optimization to the code of Fig. \ref{fig:zen}}
	\label{fig:map-map}
\end{figure}


# Hastega (ヘイスガ)
\label{sec:Hastega}

Hastega \cite{ZACKY18J, Hisae19J, ZACKY19-Hastega, ZACKY19-Hastega-Medium, ZEAM-log}は，Elixir Zen Style (\figref{fig:zen})のコードを最適化してネイティブコード化するプログラミング言語処理系である．

\figref{fig:zen} のコード相当のプログラムを OpenCL \cite{OpenCL}で記述したコードを \figref{fig:codeOpenCL} に示す．Hastega の原理は，\figref{fig:zen} と \figref{fig:codeOpenCL} のコードに類似性が多く，機械的な変換で Elixir Zen Style のコードから OpenCL のコードにコンパイルできるだろう，という着想に由来する\cite{ZACKY18J}．

\begin{figure}[t]
  \centering
{\small
\begin{verbatim}
__kernel void calc(
	__global long* input,
	__global long* outoput) {
	size_t i = get_global_id(0);
	long temp = input[i];
	temp = foo(temp);
	temp = bar(temp);
	output[i] = temp;
}
\end{verbatim}
}
	\caption{図\ref{fig:zen}のコードをOpenCLのコードに手で変換したコード}
	\ecaption{A sample code of OpenCL transformed by hand from the code of Fig. \ref{fig:zen}}
	\label{fig:codeOpenCL}
\end{figure}


このようなコードは SIMD (Single Instruction Multiple Data) アーキテクチャで並列実行するのに向いている．最近の Intel CPU には SIMD 命令が備わっており， GPU も SIMD アーキテクチャに基づいていることから， CPU や GPU で高速実行できる可能性が高い．

しかも Elixir Zen Style のコードは容易に並列性を稼いだり調整したりすることができる．\figref{fig:zen}のコードは，最大1,000,000並列で実行できることが明白である．`Enum.map`の中に記述されている処理は，互いに依存関係や副作用がないことから，並列に実行してもかまわないことが読み取れる．このような依存関係や副作用の解析は，`Enum.map`の中も Elixir Zen Style で記述する限りでは容易であると考えられる．しかも，1,000,000並列の処理を実際のSIMDコアにどのように割り当てるかについては，とくにそのことを指定したり依存したりするような記述がないことから，処理系に自由な裁量が与えられていると考えて良い．

Hastega によるコード生成の前段階で，map-mapフュージョンをあらかじめ適用しておくことが望ましい．その理由は，SIMDコアに分配して収集する処理が性能上のボトルネックになるためである．我々はそのことを線形回帰のHastegaプロトタイプの実験によって確認した\cite{Hisae19J, ZACKY19-Hastega-Medium}．

\begin{figure}[t]
  \centering
{\small
\begin{verbatim}
defmodule M do
	require Hastega
	import Hastega

	defhastega do
		def func(list) do
			list
			|> Enum.map(& &1 * 2)
			|> Enum.map(& &1 + 1)
		end
		hastegastub
	end
end
\end{verbatim}
}
	\caption{Hastega の構文}
	\ecaption{Syntax of Hastega}
	\label{fig:defhastega}
\end{figure}

我々が現在考えている Hastega は\figref{fig:defhastega}のような構文である．Hastega は，`defhastega` の中の `do` ブロックで囲まれた領域を，最適化コンパイラ部に渡す．最適化コンパイラ部では，関数定義を読み取った後， `hastegastub` により呼び出し元の関数を書き換えたコードとスタブコード，最適化されたネイティブコードを生成する．

Hastegaを開発するにあたって，他に転用できる可能性がある次の2つのライブラリを切り離す設計を採用した:

* SumMag (サムマグ): 一連のパイプラインで繋がった1つ以上の Enum.map の呼び出しを行うコードを抽出して独立した関数として定義し，元のコードを変更して，その関数を呼び出すように書き換えるメタプログラミングライブラリ
* Magicite (マジサイト): Elixir-LLVM バインディング

Hastega，SumMag，Magicite の名の由来もファイナルファンタジーからである．ファイナルファンタジーでは，ヘイスガは仲間を高速化する魔法の中でも最強の部類に相当する魔法として描かれていた．これにあやかり，Hastega という名称には，計算機を高速化する「魔法」のような存在でありたいという願いを込めている．SumMagは，ゲーム中で「召喚魔法(Summon Magic)」を表す略語である．ヘイスガは召喚魔法ではないが，後に開発するライブラリに幻獣の名称をつけようと考えているため，召喚と魔法の両方に関係する名称をつけた．メタプログラミングという召喚魔法に相当するような強力な技術を使いこなしたいという思いもある．また，Magiciteは，魔石のことである．

# Elixir マクロ
\label{sec:macro}

Elixir マクロは，Jos\'{e}が開発したメタプログラミング機構である．

Elixir では，タプルとリストで構成されるASTを操作することでメタプログラミングを実現する．したがって，LISP\cite{McCarthy:1960:RFS:367177.367199}にとてもよく似ている．

Elixir マクロでできることは，LISPとほぼ同様であり，やらないほうがいいこともLISPと似ている．とくに Elixir でマクロを使うと可読性が極めて低下するので，マクロでないとできない場合以外ではマクロを使うべきではないというソフトウェア工学的な原則が言われているが，それもまた LISP と同じ傾向である．

Elixir マクロの基本構文は次の2つである:

* プログラムコードからASTを生成する `quote/2`
* ASTに新しいコードや値を挿入する `unquote/1`

\begin{figure*}[t]
  \centering
{\small
\begin{verbatim}
 quote do: 1..1_000_000 |> Enum.map(& &1 * 2)
{:|>, [context: Elixir, import: Kernel],
 [
   {:.., [context: Elixir, import: Kernel], [1, 1000000]},
   {{:., [], [{:__aliases__, [alias: false], [:Enum]}, :map]}, [],
    [{:&, [], [{:*, [context: Elixir, import: Kernel], [{:&, [], [1]}, 2]}]}]}
 ]}
\end{verbatim}
}
	\caption{quote/2の実行例}
	\ecaption{An example of execution results of quote/2}
	\label{fig:quote}
\end{figure*}

`quote/2`の実行例を\figref{fig:quote}に示す．これは，次のようなElixir Zen Style であるコード片のASTを生成している:

* `& &1 * 2`は，第1引数の値を2倍した値を返す無名関数である．したがって，`1..1_000_000 |> Enum.map(& &1 * 2)` によって，1から1,000,000までの値それぞれを2倍したリストを生成するプログラム片を表す．
* `{}`で囲われた領域はタプルである．ASTのノードは3つの要素を持つタプルで表す．第1要素はオペレータを表し，第2要素は実行時間環境，第3要素は子ノードのリストを表している．
* `[]`で囲われた領域はリストである．
* ASTのルートのオペレータは`:|>`であるが，これはパイプライン演算子`|>`を表している．先頭の`:`はアトムであることを示している．
* ASTのルートの第2要素の`[context: Elixir, import: Kernel]`は，キーワードリストというリストの特別な場合である．キーワードリストはElixirにおいて写像を表す一表現方法である．`context`という名のアトムをキーにして，`Elixir` モジュールが対応していて，`import`という名のアトムをキーにして `Kernel`モジュールが対応している．`Elixir` モジュールと`Kernel`モジュールは，どちらもElixirが標準で提供するモジュールである．これにより，パイプライン演算子の実行時環境は，`Elixir`モジュールの中で実行されているというコンテキストで実行されており，その中で`Kernel`モジュールがインポートされているということを意味する．\figref{fig:quote}は，Elixirの対話シェルである`iex`コマンド上で実行したので，このような実行時環境となっている．

ASTのルートの1番目の子ノード`{:.., ..., [1, 1000000]}`は，範囲演算子`..`を使った記述 `1..1_000_000` を表している．このノードの1番目の子ノードが1，2番目の子ノードが1,000,000である．このようにASTのノードは3つの要素を持つタプルの他に，直接値を表すこともある．内部表現では整数の中の`_`は無視されている．

ASTのルートの2番目の子ノードは，`Enum.map(& &1 * 2)`を表している．その本体は，次のようなASTノードの3つ組である:

* 第1要素は，演算子`.`である．その実行時環境は空である．1番目の子ノードが`Enum`モジュール，2番目の子ノードが　`map` 関数である．
* 第2要素の実行時環境は空である．
* 子ノードとして，演算子`&`を持つ．

演算子`&`のノードは，無名関数を表す．

* その実行時環境は空である．
* 子ノードとして，乗算演算子`*`を持つ．
	* その実行時環境は`iex`上の環境である．
	* 第1子ノードは演算子`&`であり，その子ノードは1である．これにより`&1`すなわち第1引数を表す．
	* 第2子ノードは2である．
	* これにより，第1引数に2を乗じたものを表す．

`unquote/1`は`quote/2`の中でのみ使用できる．

# SumMag (サムマグ)の設計
\label{sec:design}


# まとめと将来課題
\label{sec:summary}