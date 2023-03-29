---
title: "lizardを使ってCCNというコード品質の指標を学ぶ"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CCN", "lizard", "pytest"]
published: true
published_at: 2023-03-30 09:00
---

コード解析の仕組みやツールを調べていたら、lizardというツールを発見しました。
複数の言語をサポートしていてPythonライブラリとして使用可能で、しかも個人開発のOSSなのにソフトウェア工学の研究でも度々引用されていて...実用性と自身の関心を両立できそうだったので教材として使い始めました。

手を動かしつつ、Wikipediaの[Cyclomatic_complexit](https://en.wikipedia.org/wiki/Cyclomatic_complexity)解説を少し理解できれば良いな、という動機です。


## lizardによる単体テスト例
最初にサンプルコードですが、プロジェクト直下のソースコード全てを評価対象としています。テストコードとして簡単に実装できることは確認できました。

```python
class TestLizard(TestCase):
    def lizard_targets(self):
        path = './*.py'
        files = glob.glob(path)
        return files

    def test_lizard(self):
        targets = self.lizard_targets()
        if not targets:
            assert True
        else:
            results = []
            for code in targets:
                analysis = lizard.analyze_file(code)
                for i in analysis.function_list:
                    result = {}
                    result['name'] = analysis.__dict__['filename'] + '.' + i.__dict__['name']
                    result['ccn'] = i.__dict__['cyclomatic_complexity'] 
                    result['nloc'] = i.__dict__['nloc']

                    assert result['ccn'] < 15 # cyclomatic complexity number
                    assert result['nloc'] < 50 # lines of code without comments
                    results.append(result)

            print(results)

# 参考: print()結果
tests/test_**.py .[{'name': './20230106.py.main', 'ccn': 8, 'nloc': 21}, {'name': './loops_multiple_edges.py.compare_links', 'ccn': 4, 'nloc': 9}, {'name': './loops_multiple_edges.py.main', 'ccn': 7, 'nloc': 17}, {'name': './2021_q3_token_passing.py.main', 'ccn': 8, 'nloc': 23}]
```

今回の評価対象は手元にあったpaizaの回答コードを使っており、さすがに複雑度を表す`ccn`は低い数値になってます。

CCNとは何を計測している指標なのか、分かっていないまま動かせるのはプログラムの偉大なところですが...この指標をもとに何を判断できるのか、lizardの実装を調べました。


## lizardの実装方針
> This tool actually calculates how complex the code 'looks' rather than how complex the code really 'is'. People will need this tool because it's often very hard to get all the included folders and files right when they are complicated. But we don't really need that kind of accuracy for cyclomatic complexity.

上記の通り、READMEに明記されていますが見かけ（制御構文の多さ）を重視して、コード自体のロジック（言い換えればコンテクスト）は考慮せずに複雑度を評価する、と言っています。

CCNは関数・メソッド単位で評価される指標ですが、クロージャのような実装をしてもlizardによるCCN値は増加しないはずで、仕様を理解して使う分には割り切りが必要になりそうです。

例えば、lizardのCCN値をテストケース数の指標として使いたい場合、lizardでは関数同士・クラス間の依存関係は全く考慮されないということになります。

ある関数内のコードがどの程度複雑でテストコードを実装しやすい/しづらい（ように見える）か、を定量的に評価するツールになりそうです。

## lizardソースコード: トークン毎のCCN評価

https://github.com/terryyin/lizard/blob/master/lizard.py#L531-L536

ソースコードを`token`に分割し、「tokenがconditionに一致する場合はCCNの値を+=1してインクリメントする」という簡潔な実装になっています。

そして`token`（特定の記号、改行、空白行やコメントなど）と`condition`（CCNの増加条件、つまり分岐の記法）は言語依存でパターンマッチするよう、`lizard_languages/**.py`のReaderクラスに実装されています。

Pythonでは`condition`の指定はこんな感じでした。
各言語でハードコードされていて、構文解析をしない代わりにシンプルさを維持しつつ実用性を重視している印象です。

```python
    # PythonReaderクラス抜粋
    _conditions = set(['if', 'for', 'while', 'and', 'or',
                      'elif', 'except', 'finally'])
```

```python
    # PythonReaderクラス抜粋
    @staticmethod
    def generate_tokens(source_code, addition='', token_class=None):
        return ScriptLanguageMixIn.generate_common_tokens(
                source_code,
                r"|\'\'\'.*?\'\'\'" + r'|\"\"\".*?\"\"\"', token_class)

    # ScriptLanguageMixInクラス抜粋
    @staticmethod
    def generate_common_tokens(source_code, addition, match_holder=None):
        _until_end = r"(?:\\\n|[^\n])*"
        return CodeReader.generate_tokens(
            source_code,
            r"|\#" + _until_end + addition,
            match_holder)
```

`token`のパースは言語共通クラス`CodeReader`を各言語から呼び出していて、ややボリュームあります。
:::details CodeReader.generate_tokens

https://github.com/terryyin/lizard/blob/master/lizard_languages/code_reader.py#L108-L157
:::

## CCNの目的と使うタイミングは？
CCNはプログラムの実行パスを数えているので「保守性の容易さ」の指標となっている。理想的にはテストコードのケース数として看做せるため「リファクタリングする前にテストコードを書く必要性を認識できる」という意見も見かけた。

プログラムのコンテクストを重視しすぎると計算量（処理性能）までテストしたくなりそうなので、個人的にはCCN指標の実装としてlizardは仕事でも運用したいところです。「この閾値を超えたら単体テスト必須」くらいの始め方でも良いかもしれない。開発目的としては「変更容易性と保守性を両立する」ことなので、使うなら開発初期からCCN計測を始めると相対的にも評価できるので良いはず。

### 参考ページ

https://blog.feabhas.com/2018/07/code-quality-cyclomatic-complexity/
>  at least with a low CCN you have a greater chance of deciphering the intent of the function’s behaviour and possibly being able to write a set of unit tests before refactoring it.
>> 少なくともCCNが低ければ、関数の動作の意図を読み解くことができ、リファクタリングする前にユニットテストのセットを書くことができる可能性がある

https://qiita.com/kijuky/items/fa9f9eedfda9ea716106
> 要約すると
> lizard は CCN を正確には計算しない。
> さまざまな言語の複雑度の計算を簡単な実装で実現するために、パーサーとして実装する。


https://www.infoq.com/news/2008/03/cyclomaticcomplexity/
> The Enerjy results have created some confusion around how CC is counted. Keith Braithwaite pointed out that Enerjy study counted CC at the file level, not the method level. Christopher Beck, commenting on The Quality of TDD, chimed in saying:
> ... it’s not CC (and shouldn’t be named CC). Rather it comes close to another metric called WMC or “Weighted Methods per Class”, which sums up CC values of a class.
>> Enerjyの結果は、CCのカウント方法についていくつかの混乱を引き起こした。Keith Braithwaiteは、Enerjyの研究ではCCをメソッドレベルではなく、ファイルレベルでカウントしていると指摘しました。The Quality of TDDにコメントしているChristopher Beckは、次のようにコメントした;
>> ...それはCCではない（そしてCCと名付けるべきでもない）。むしろ、クラスのCC値を合計するWMC（Weighted Methods per Class）と呼ばれる別の指標に近いと言えるでしょう。
