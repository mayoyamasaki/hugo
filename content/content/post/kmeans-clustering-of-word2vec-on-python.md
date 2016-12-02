+++
date = "2016-11-18T00:04:32+09:00"
slug = "kmeans clustering of word2vec on python"
tags = ["ML", "NLP", "Python"]
title = "K-Means Clustering of Word2Vec on Python"
description = "scikit-learnのK-Means実装を使って、学習済みWord2Vecのクラスタリングを行った。"

+++

## 概要
- scikit-learnのK-Means実装を使って、学習済みWord2Vecのクラスタリングを行った。
- それなりに上手く、クラスタリングできていそうだった。

## はじめに
ACL2014で、EmbeddingsのクラスタをNER(Named Entity Recognition)に使用している論文がある[^1]。  
線形モデルには、低次元連続値の素性(特徴量)より、高次元離散値の素性が良いらしい。  
この記事では、Word2Vecで学習した単語ベクトル表現(連続値)を使って、K-Meansによるクラスタリング(離散値)を行ってみる。

## 設定
- [Google Code word2vec](https://code.google.com/archive/p/word2vec/)にて公開されている、GoogleNews-vectors-negative300.bin.gzを入力に用いた。
- PyPIで公開されている[word2vec](https://github.com/danielfrg/word2vec)がTypeErrorで上手くモデルファイルをロードできなかったので、gensimの実装を使った。
- K-Meansには、データ量が大きかったので、scikit-learnの[MiniBatchKMeans](http://scikit-learn.org/stable/modules/generated/sklearn.cluster.MiniBatchKMeans.html#sklearn.cluster.MiniBatchKMeans.fit)を用いた[^2]。

学習済みのWord2Vecのモデルファイルはいろいろ公開されていて、3Topというstartupのまとめがよかった[^3]。  
gensimのWord2Vec実装のAPIドキュメント[^4]は、あんまり丁寧に書かれていないので、結構つらかった。  

## 実装

```python
import argparse
from functools import lru_cache
from logging import getLogger

import numpy as np
from gensim.models.word2vec import Word2Vec
from sklearn.cluster import MiniBatchKMeans
from sklearn.externals import joblib


logger = getLogger(__name__)


def make_dataset(model):
    """Make dataset from pre-trained Word2Vec model.
    Paramters
    ---------
    model: gensim.models.word2vec.Word2Vec
        pre-traind Word2Vec model as gensim object.
    Returns
    -------
    numpy.ndarray((vocabrary size, vector size))
        Sikitlearn's X format.
    """
    V = model.index2word
    X = np.zeros((len(V), model.vector_size))

    for index, word in enumerate(V):
        X[index, :] += model[word]
    return X


def train(X, K):
    """Learn K-Means Clustering with MiniBatchKMeans.
    Paramters
    ---------
    X: numpy.ndarray((sample size, feature size))
        training dataset.
    K: int
        number of clusters to use MiniBatchKMeans.
    Returens
    --------
    sklearn.cluster.MiniBatchKMeans
        trained model.
    """
    logger.info('start to fiting KMeans with {} classs.'.format(K))
    classifier = MiniBatchKMeans(n_clusters=K, random_state=0)
    classifier.fit(X)
    return classifier


def main():
    parser = argparse.ArgumentParser(
        description='Python Word2Vec Cluster')

    parser.add_argument('model',
                        action='store',
                        help='Name of word2vec binary modelfile.')

    parser.add_argument('-o', '--out',
                        action='store',
                        default='model.pkl',
                        help='Set output filename.')

    parser.add_argument('-k', '--K',
                        action='store',
                        type=int,
                        default=500,
                        help='Num of classes on KMeans.')

    parser.add_argument('-p', '--pre-trained-model',
                        action='store',
                        default=None,
                        help='Use pre-trained KMeans Model.')

    parser.add_argument('-w', '--words-to-pred',
                        action='store',
                        nargs='+',
                        type=str,
                        default=None,
                        help='List of word to predict.')

    args = parser.parse_args()

    model = Word2Vec.load_word2vec_format(args.model, binary=True)

    if not args.pre_trained_model:
        X = make_dataset(model)
        classifier = train(X, args.K)
        joblib.dump(classifier, args.out)
    else:
        classifier = joblib.load(args.pre_trained_model)

    if args.words_to_pred:

        X = [model[word] for word in args.words_to_pred if word in model]
        classes = classifier.predict(X)

        result = []
        i = 0
        for word in args.words_to_pred:
            if word in model:
                result.append(str(classes[i]))
                i += 1
            else:
                result.append(str(-1))
        print(' '.join(result))


if __name__ == '__main__':
    main()
```

実装は、Githubに公開している通りだが、Word2Vecの辞書をNumpyのndarrayに変換するやり方が、最適ではない気がする。まあ一度だけしか実行しないので、今回は気にしない[^5]。

## 使い方

### 学習

```sh
$ python3 w2vcluster/w2vcluster.py GoogleNews-vectors-negative300.bin -k 500 -o model1000.pkl
```

### 予測

```sh
python3 w2vcluster/w2vcluster.py GoogleNews-vectors-negative300.bin -p model500.pkl -w apple Apple banana Google
176 118 176 118
```

argparseの可変長リストは初めて使った。見ての通り、appleとbananaはクラスタID 176に、AppleとGoogleはクラスタID 118に分類されており、上手くいっているような気がする。
モデルファイルは、joblibのpickleファイルなので、もちろんPythonからも使うことができる。やり方、READMEに一応書いている。

## おわりに
MiniBatchKMeansを使うと、大規模データでもMacbook Proで問題なく計算できた(数分程度で終わるが、通常のKMeansだと終わる気配がなかった...)。  
離散化したWord2Vecを何らのモデルでつかってみたい。ソフトクラスタリングも試してみたい。


[^1]:[Jiang Guo et al. Revisiting Embedding Features for Simple Semi-supervised Learning](http://aclweb.org/anthology/D/D14/D14-1012.pdf)
[^2]:[Comparison of the K-Means and MiniBatchKMeans clustering algorithms](http://scikit-learn.org/stable/auto_examples/cluster/plot_mini_batch_kmeans.html#sphx-glr-auto-examples-cluster-plot-mini-batch-kmeans-py)
[^3]:[word2vec-api](https://github.com/3Top/word2vec-api)
[^4]:[models.word2vec – Deep learning with word2vec](https://radimrehurek.com/gensim/models/word2vec.html)
[^5]:[Python K-Means Cluster of Word2Vec](https://github.com/mayoyamasaki/py-kmeans-word2vec)
