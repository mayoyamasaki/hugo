+++
date = "2016-11-16T00:29:37+09:00"
slug = "introduction-to-neo4j"
tags = ["Neo4j", "Python"]
title = "Introduction to Neo4j"
description = "- Neo4jの公式ドキュメントを読んだので、概要をまとめた。概念、クエリ文法、Pythonドライバの使い方について解説した。"
+++

## 概要

- Neo4jの公式ドキュメントを読んだので、概要をまとめた。
- 概念、クエリ文法、Pythonドライバの使い方について解説した。

## はじめに
![Neo4j](/images/introduction-to-neo4j/neo4j_logo.png)

2016/11/15現在、Neo4jの最新版は、3系(3.0.7)で、公式サイトからdmg(Mac OSの場合)が配布されている[^1]。  
公式ドキュメントのv3を斜め読みしたので、簡単に解説する[^2]。

## 導入
Neo4jはスキーマレスDBで、グラフデータの扱い特化することで、高い検索パフォーマンスを発揮している。
データは、「Node」、「Relationship」、「Propetries」で表現される。  
下図の場合、PersonやMovieで囲まれいるオブジェクトがNodeで、Node間に貼られているエッジ(有効辺の場合は矢印)がRelationshipであり、各Nodeが所有するnameやbornなどの値がPropetriesである。

![Example Graph](/images/introduction-to-neo4j/graphdb-simple-labels.svg)

またNodeやRelationshipには、「Label」と呼ばれる複数個付与することができる。Labelは何らかの集合を意味し、上図の場合は、PersonやMovie、ACTED_INがLabelに相当する。

データはこれらの概念で表現されるが、グラフを探索する上で、「Path」と「Traversal」という概念がある。  
Pathは、Relationshipで接続されるNodeのことで、検索結果などを表す概念である。また、Traversalは検索方法を表す概念であり、「Tom HanksというNodeとACTED_INのRelationshipにあるNode」というようなクエリのことを意味している。この場合、Forrest GumpのNodeが下図のようなPathとして返されることになる。

![Example Path](/images/introduction-to-neo4j/graphdb-path.svg)

## 入門

### Cypher
Neo4jのデータは、Cypherと呼ばれるクエリ言語用いて扱う。例えば、Pathの例であげた図のデータをCypherで表現すると次の様になる(tomやgumpはアクセス用の変数である)。

```c
(tom:Person     {name: "Tom Hanks", born:1986})
-[role:ACTED_IN {roles: ["Forrest"]}] ->
(gump:Movie     {title: "Forrest Gump", released:1994})
```

### データの作成と変数
CREATE文でデータの作成、RETURNで変数からデータを取得できる。

```c
CREATE (tom:Person     {name: "Tom Hanks", born:1986})
-[role:ACTED_IN {roles: ["Forrest"]}] ->
(gump:Movie     {title: "Forrest Gump", released:1994})
RETURN tom
```

### MATCHによるデータの検索
データ表現をMATCHに渡すことで、複雑なクエリも構築できる。

```c
MATCH (p:Person { name:"Tom Hanks" })-[r:ACTED_IN]->(m:Movie)
RETURN m.title, r.roles
```

### データのマージ
データやPropetriesが存在しないときだけ、その値を追加したい場合はMERGEが使える。

```c
MERGE (m:Movie { title:"Cloud Atlas" })
ON CREATE SET m.released = 2012
RETURN m
```

### 結果のフィルタリング
RDBのSQLと同じく、WHEREによる条件指定やASによるエイリアス、ORDER BYによるソートなどの機能がある[^3]。


## ドライバ
公式ドキュメントでは、C#、Java、JavaScrip、Pythonのドライバを紹介している[^4]。

```python
from neo4j.v1 import GraphDatabase, basic_auth

driver = GraphDatabase.driver(
        "bolt://localhost", auth=basic_auth("neo4j", "neo4j"), encrypted=True)

session = driver.session()

session.run("CREATE (a:Person {name:'Arthur', title:'King'})")

result = session.run("MATCH (a:Person) WHERE a.name = 'Arthur' "
                      "RETURN a.name AS name, a.title AS title")

for record in result:
    print("%s %s" % (record["title"], record["name"]))

session.close()
```

これは、Pythonのサンプルプグラムのそのままであるが、このコードの理解だけで、問題なく利用できる。  
さて、boltスキーマを初めて見る人が多いと思うが、Neo4jが策定したバイナリネットワークプトトコルであるとのこと[^5]。

## おわりに
とりあえず基本的な内容は理解できたが、Cypherはパワフルなクエリ言語なので、全容はつかめていないが、とりあえず使う文には、難しくない様に感じた。  
今後は、Neo4jのグラフデータを可視化するツールはいくつかあり、[Popotojs](http://www.popotojs.com/)が良さそうなので、これを試してみたい。


[^1]: [neo4j](https://neo4j.com/)
[^2]: [The Neo4j Developer Manual v3.0](https://neo4j.com/docs/developer-manual/current/)
[^3]: [Chapter 2. Get started](https://neo4j.com/docs/developer-manual/current/get-started/)
[^4]: [Chapter 4. Drivers](https://neo4j.com/docs/developer-manual/current/drivers/)
[^5]: [Neo4j 3.0がリリース，バイナリ通信プロトコルと標準ドライバを装備](https://www.infoq.com/jp/news/2016/05/neo4j-3.0)
