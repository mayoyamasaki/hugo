+++
date = "2016-11-04T21:52:39+09:00"
slug = "html-to-text-for-nlp"
tags = ["Python", "NLP"]
title = "HTML to TEXT for NLP"
description = "「HTMLからテキストを取り出す」というタスクは、一見簡単の様で難しい。特に、後続するテキスト処理(文分割や形態素解析、構文解析)を上手くしようとすると難しい。ここでは、英語文書に対して、ナイーブなルールで、テキスト抽出をそこそこ上手くやる方法を紹介する。"

+++

## 概要

「HTMLからテキストを取り出す」というタスクは、一見簡単の様で難しい。  
特に、後続するテキスト処理(文分割や形態素解析、構文解析)を上手くしようとすると難しい。  
ここでは、英語文書に対して、ナイーブなルールで、テキスト抽出をそこそこ上手くやる方法を紹介する。

## はじめに

この記事では、Github上で公開している実装の簡単な紹介をする[^1]。

[^1]: https://github.com/mayoyamasaki/html2text

使い方は、以下のようにコマンドラインツールとして使ったり、

```sh
python3 html2text.py https://example.com > example.com.txt
```

Pythonを通して利用できる。

```sh
$ python3
>>> import html2text
>>> html = html2text.url2html('https://example.com')
>>> bodyhtml = html2text.html2body(html)
>>> text = html2text.html2text(bodyhtml)
```

## 実装

### スペースの統一
タブ文字、[NBSP](https://ja.wikipedia.org/wiki/%E3%83%8E%E3%83%BC%E3%83%96%E3%83%AC%E3%83%BC%E3%82%AF%E3%82%B9%E3%83%9A%E3%83%BC%E3%82%B9)```\xa0```等を半角スペースに置換する。
NBSPは、よく構文解析などが例外を吐くので注意。

### ブロックタグ削除の影響
```foo.</p><p>bar```の様なケースでは、単に削除すると、```foo.bar```となり、TokenizeやSentence segmentationで失敗する原因になる。また```foo.</div>bar```の様なケースもあるので、単にブロックタグの連接だけでなく、ブロックタグの直後に空白や改行無しに、文字が連接するかをチェックする必要がある。  
ここでは、前者に対しては改行、後者に対してはスペースを挿入することで対処している。

### タグ削除による影響
ブロックタグに限らず、CSSでblockやinline-block等をしていしているインラインタグでも、同種の影響がでる。もっと言えば、HTML

