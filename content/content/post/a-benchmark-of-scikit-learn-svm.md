+++
date = "2016-11-12T18:41:26+09:00"
slug = "a-benchmark-of-scikit-learn-svm"
tags = ["ML", "Python"]
title = "A Benchmark of scikit-learn SVMs"
description = "Baggingと多項式カーネルを使ったSVMが、精度と学習及び予測速度の観点で、良さそうであった。"
+++

## 概要
- Baggingと多項式カーネルを使ったSVMが、精度と学習及び予測速度の観点で、良さそうであった。

## はじめに
あるシステムの実行速度が遅く、プロファイルしてみたところ、SVMがボトルネックになっていた。
他の処理(別の学習器も含む)と比較しても数十〜数百倍、学習と推定が遅かったので、[scikit-learn](http://scikit-learn.org/stable/)のSVMのベンチマークを取ってみた。

## 設定

- SVM実装は、SVC(rbf, poly), LinearSVCを使用した。
- OneVsRestの多クラス分類で、irisデータセットを使用した。
- それぞれについて、BaggingClassifierを使ったアンサンブルを試した。
- ついでに、RandomForestClassifierも試した。

## 実装

```python
import time
from collections import defaultdict
from itertools import cycle

import matplotlib.pyplot as plt
import numpy as np
from sklearn.ensemble import BaggingClassifier
from sklearn.ensemble import RandomForestClassifier
from sklearn import datasets
from sklearn.multiclass import OneVsRestClassifier
from sklearn.svm import SVC
from sklearn.svm import LinearSVC
from sklearn.metrics import f1_score
from sklearn.model_selection import train_test_split


def load_dataset():
    X, y = datasets.load_iris(return_X_y=True)
    X = np.repeat(X, 300, axis=0)
    y = np.repeat(y, 300, axis=0)
    return X, y


def timeit(func, args):
    start = time.clock()
    results = func(*args)
    end = time.clock()
    return results, end-start


def make_clfs():
    classifiers = []
    SVCs = (
        (SVC(kernel='linear'), 'SVC,linear'),
        (SVC(kernel='rbf'), 'SVC,rbf'),
        (SVC(kernel='poly'), 'SVC,poly'),
    )
    classifiers.extend(
        [(OneVsRestClassifier(estimater), 'OvR'+title)
            for estimater,title in SVCs])
    classifiers.extend(
        [(OneVsRestClassifier(
            BaggingClassifier(estimater, max_samples=0.1)), 'OvR,Bagging,'+title)
                for estimater,title in SVCs])
    classifiers.append((LinearSVC(), 'LinearSVC'))
    classifiers.append((RandomForestClassifier(), 'RandomForest'))
    return classifiers


def sample(X, y, train_size):
    if train_size == 1.: return X, y
    _X, _, _y, _  = train_test_split(X, y, train_size=train_size, random_state=0)
    return _X, _y


def plot(graph_title, train_sizes, dic):
    colors = cycle(['b', 'g', 'r', 'c', 'm', 'y', 'k', 'w'])
    plt.figure()
    plt.title(graph_title)
    plt.xlabel("Training examples")
    plt.ylabel(graph_title)
    plt.grid()

    for label, vals in dic.items():
        plt.plot(train_sizes, vals, 'o-', color=next(colors), label=label)
    plt.legend(loc="best")
    return plt


def main():
    CV=5
    X, y = load_dataset()
    classifiers = make_clfs()
    train_sizes = np.linspace(.1, 1., 5)
    fit_times = defaultdict(list)
    pred_times = defaultdict(list)
    fscores = defaultdict(list)

    for train_size in train_sizes:
        _X, _y = sample(X, y, train_size)
        for clf, title in classifiers:
            scores = {'fit':[], 'pred':[], 'fscore':[] }
            for n_cv in range(CV):
                X_train, X_test, y_train, y_test = train_test_split(
                    _X, _y, test_size=0.4, random_state=0)

                _, t = timeit(clf.fit, (X_train, y_train))
                scores['fit'].append(t)

                y_pred, t = timeit(clf.predict, (X_test, ))
                scores['pred'].append(t)

                fscore = f1_score(y_test, y_pred, average='macro')
                scores['fscore'].append(fscore)

            fit_times[title].append(np.mean(scores['fit']))
            pred_times[title].append(np.mean(scores['pred']))
            fscores[title].append(np.mean(scores['fscore']))

    plot('Fscores', train_sizes, fscores)
    plot('Time of Training', train_sizes, fit_times)
    plot('Time of Predication', train_sizes, pred_times)
    plt.show()


if __name__ == '__main__':
    main()
```

```make_clfs```で作成した分類器一つ一つに対して、学習時間、推測時間、F値(マクロ平均)を求めている。
尚、これらの値は5回施行の平均値であり、データ量増加に対する、推移を見ている。


## 結果

### F値

![fscores](/images/a-benchmark-of-scikit-learn-svm/fscores.png)

一瞬見失うが、RandomForestの精度が最も高く、非線形カーネル、線形カーネルの順番に精度が高かった。
また、Baggingを加えても非線形カーネルではほとんど差が出ていない。

### 学習時間
![trains](/images/a-benchmark-of-scikit-learn-svm/trains.png)
多項式カーネルでは、Baggingの効果が顕著にでている。また、SVCの線形カーネルは話に聞く通り低パフォーマンスで、LinearSVCの方が学習時間は少ない。  

> 単位記述を忘れたが、縦軸は[s]である。

### 予測時間
![preds](/images/a-benchmark-of-scikit-learn-svm/preds.png)
予測時間では、一転してBaggingを使用しない方が速度がでている傾向にある。また、推測においてもSVCの線形カーネルのパフォーマンスは低い。

## おわりに
この手のベンチマークはデータセットに依存するが、今回の評価では、SVMの中では、Baggingを使った多項式カーネルによるOneVsRestが有効であった。  
今回の様に汎化しやすそうなデータセットでは、やはりRandomForestは有効で、今後は、多様なデータセットでの検証をやってみたい。  
また、Baggingやアンサンブル学習を初めて試してみたが、実行速度改善に大きく寄与したので、実システムでも試してみたい。尚、Baggingについは、アンサンブル学習のスライドが分かりやすかった[^1]。

[^1]: [アンサンブル学習](http://www.slideshare.net/holidayworking/ss-11948523)


