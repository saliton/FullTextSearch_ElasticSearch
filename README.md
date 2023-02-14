
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/saliton/FullTextSearch_ElasticSearch/blob/main/FullTextSearch_ElasticSearch.ipynb)

# Colabで全文検索（その６：ElasticSearch編）

各種全文検索ツールをColabで動かしてみるシリーズです。全7回の予定です。今回はElasticSearchです。

処理時間の計測はストレージのキャッシュとの兼ね合いがあるので、2回測ります。2回目は全てがメモリに載った状態での性能評価になります。ただ1回目もデータを投入した直後なので、メモリに載ってしまっている可能性があります。

## 準備

まずは検索対象のテキストを日本語wikiから取得して、Google Driveに保存します。（※ Google Driveに約１GBの空き容量が必要です。以前のデータが残っている場合は取得せず再利用します。）

Google Driveのマウント


```python
from google.colab import drive
drive.mount('/content/drive')
```

    Mounted at /content/drive


jawikiの取得とjson形式に変換。90分ほど時間がかかります。他の全文検索シリーズでも同じデータを使うので、他の記事も試す場合は wiki.json.bz2 を捨てずに残しておくことをおすすめします。


```shell
%%time
%cd /content/
import os
if not os.path.exists('/content/drive/MyDrive/wiki.json.bz2'):
    !wget https://dumps.wikimedia.org/jawiki/latest/jawiki-latest-pages-articles.xml.bz2
    !pip install wikiextractor
    !python -m wikiextractor.WikiExtractor --no-templates --processes 4 --json -b 10G -o - jawiki-latest-pages-articles.xml.bz2 | bzip2 -c > /content/drive/MyDrive/wiki.json.bz2
```

    /content
    CPU times: user 2.15 ms, sys: 0 ns, total: 2.15 ms
    Wall time: 2.6 ms


json形式に変換されたデータを確認


```python
import json
import bz2

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    for n, line in enumerate(fin):
        data = json.loads(line)
        print(data['title'].strip(), data['text'].replace('\n', '')[:40], sep='\t')
        if n == 5:
            break
```

    アンパサンド	アンパサンド（&amp;, ）は、並立助詞「…と…」を意味する記号である。ラテン
    言語	言語（げんご）は、広辞苑や大辞泉には次のように解説されている。『日本大百科事典』
    日本語	 日本語（にほんご、にっぽんご）は、日本国内や、かつての日本領だった国、そして日
    地理学	地理学（ちりがく、、、伊：geografia、）は、。地域や空間、場所、自然環境
    EU (曖昧さ回避)	EU
    国の一覧	国の一覧（くにのいちらん）は、世界の独立国の一覧。対象.国際法上国家と言えるか否


## ElasticSearchのインストール

### ダウンロード


```bash
%%bash

wget -q https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.9.2-linux-x86_64.tar.gz
wget -q https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.9.2-linux-x86_64.tar.gz.sha512
tar -xzf elasticsearch-oss-7.9.2-linux-x86_64.tar.gz
sudo chown -R daemon:daemon elasticsearch-7.9.2/
shasum -a 512 -c elasticsearch-oss-7.9.2-linux-x86_64.tar.gz.sha512 
```

    elasticsearch-oss-7.9.2-linux-x86_64.tar.gz: OK


### Kuromojiのインストール

Kuromojiを導入しま、、、せん。

今回は他と揃えるため、ngramを使うことにします。

```shell
#!sudo elasticsearch-7.9.2/bin/elasticsearch-plugin install analysis-kuromoji
```

## Elasticsearchの起動

バックグラウンドでElasticsearchを起動します。

```bash
%%bash --bg

sudo -H -u daemon elasticsearch-7.9.2/bin/elasticsearch
```

    Starting job # 0 in a separate thread.


起動を少し待ちます。


```python
import time
# Sleep for few seconds to let the instance start.
time.sleep(20)
```

プロセスを確認します。


```shell
!ps -ef | grep elasticsearch | grep -v grep
```

    root         282     280  0 21:46 ?        00:00:00 sudo -H -u daemon elasticsearch-7.9.2/bin/elasticsearch
    daemon       283     282 99 21:46 ?        00:00:19 /content/elasticsearch-7.9.2/jdk/bin/java -Xshare:auto -Des.networkaddress.cache.ttl=60 -Des.networkaddress.cache.negative.ttl=10 -XX:+AlwaysPreTouch -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -XX:-OmitStackTraceInFastThrow -XX:+ShowCodeDetailsInExceptionMessages -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dio.netty.allocator.numDirectArenas=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Djava.locale.providers=SPI,COMPAT -Xms1g -Xmx1g -XX:+UseG1GC -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -Djava.io.tmpdir=/tmp/elasticsearch-4888719369848925463 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=data -XX:ErrorFile=logs/hs_err_pid%p.log -Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m -XX:MaxDirectMemorySize=536870912 -Des.path.home=/content/elasticsearch-7.9.2 -Des.path.conf=/content/elasticsearch-7.9.2/config -Des.distribution.flavor=oss -Des.distribution.type=tar -Des.bundled_jdk=true -cp /content/elasticsearch-7.9.2/lib/* org.elasticsearch.bootstrap.Elasticsearch


動作を確認します。


```shell
!curl -sX GET "localhost:9200/"
```

    {
      "name" : "c1190554431f",
      "cluster_name" : "elasticsearch",
      "cluster_uuid" : "ZogVcve2S2C1AlDmu8GWtA",
      "version" : {
        "number" : "7.9.2",
        "build_flavor" : "oss",
        "build_type" : "tar",
        "build_hash" : "d34da0ea4a966c4e49417f2da2f244e3e97b4e6e",
        "build_date" : "2020-09-23T00:45:33.626720Z",
        "build_snapshot" : false,
        "lucene_version" : "8.6.2",
        "minimum_wire_compatibility_version" : "6.8.0",
        "minimum_index_compatibility_version" : "6.0.0-beta1"
      },
      "tagline" : "You Know, for Search"
    }


## Pythonライブラリのインストール


```shell
!pip install elasticsearch==7.9.1
```

    Collecting elasticsearch==7.9.1
      Downloading elasticsearch-7.9.1-py2.py3-none-any.whl (219 kB)
    |████████████████████████████████| 219 kB 6.9 MB/s 
    Requirement already satisfied: urllib3>=1.21.1 in /usr/local/lib/python3.7/dist-packages (from elasticsearch==7.9.1) (1.24.3)
    Requirement already satisfied: certifi in /usr/local/lib/python3.7/dist-packages (from elasticsearch==7.9.1) (2021.10.8)
    Installing collected packages: elasticsearch
    Successfully installed elasticsearch-7.9.1


## データのインポート

データを50万件インポートします。

```python
import json
import bz2
from tqdm.notebook import tqdm
from elasticsearch import Elasticsearch, helpers

es = Elasticsearch(["http://localhost:9200"])
index_name = 'wiki_jp'
if es.indices.exists(index=index_name):
    res = es.indices.delete(index=index_name)
    print("Response from server: {}".format(res))
print("creating the '{}' index.".format(index_name))

index_body = {
    "settings": {
        "max_result_window": 50000,
        "analysis": {
            "tokenizer": {
                "ja_ngram_tokenizer": {
                    "type": "ngram",
                    "min_gram": 2,
                    "max_gram": 2,
                    "token_chars": [
                        "letter",
                        "digit"
                    ],
                },
            },
            "analyzer": {
                "ja_ngram_index_analyzer": {
                    "type": "custom",
                    "char_filter": [
                        "html_strip"
                    ],
                    "tokenizer": "ja_ngram_tokenizer",
                    "filter": [
                        "lowercase"
                    ]
                },
                "ja_ngram_search_analyzer": {
                    "type": "custom",
                    "char_filter": [
                        "html_strip"
                    ],
                    "tokenizer": "ja_ngram_tokenizer",
                    "filter": [
                        "lowercase"
                    ]
                }
            }
        }
    },
    "mappings": {
        "properties": {
            "title": {"type": "keyword"},
            "body": {"type": "text",
                     "search_analyzer": "ja_ngram_search_analyzer",
                     "analyzer": "ja_ngram_index_analyzer"}
        }
    }
}

res = es.indices.create(index=index_name, body=index_body)
print("Response from server: {}".format(res))
print(es.indices.get_mapping(index=index_name))

limit = 500000
def gendata():
    with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding="utf-8") as fin:
        n = 0
        for line in tqdm(fin, total=limit*1.5):
            data = json.loads(line)
            title = data['title'].strip()
            body = data['text'].replace('\n', '')
            if len(title) > 0 and len(body) > 0:
                n += 1
                yield {
                    "_op_type": "create",
                    "_index": index_name,
                    "_source": {"title": title, "body": body}
                }
            if n == limit:
                break

helpers.bulk(es, gendata())
es.close()
```
    creating the 'wiki_jp' index.
    Response from server: {'acknowledged': True, 'shards_acknowledged': True, 'index': 'wiki_jp'}
    {'wiki_jp': {'mappings': {'properties': {'body': {'type': 'text', 'analyzer': 'ja_ngram_index_analyzer', 'search_analyzer': 'ja_ngram_search_analyzer'}, 'title': {'type': 'keyword'}}}}}



      0%|          | 0/750000.0 [00:00<?, ?it/s]


登録件数を確認します。


```shell
!curl -XGET 'http://localhost:9200/wiki_jp/_count'
```

    {"count":500000,"_shards":{"total":1,"successful":1,"skipped":0,"failed":0}}

## Pythonで検索

インデックスなしでの検索方法は分からなかったので、インデックスありの検索、1回目です。


```python
import time
from elasticsearch import Elasticsearch, helpers

es = Elasticsearch(["http://localhost:9200"])
index_name = 'wiki_jp'
query = {"query" : {"match_phrase" : {"body" : "日本語"}}, "profile": "true"}
start_time = time.perf_counter()
results = es.search(index=index_name, body=query, size=20000)
print("{:.2f} sec".format(time.perf_counter() - start_time))
es.close()
```

    7.48 sec


MySQLやPostgresSQLと同等です。

結果の転送を含まない内部での検索時間は以下で取得できます。


```python
time_in_nanos = results['profile']['shards'][0]['searches'][0]['query'][0]['time_in_nanos']
print('{:.2f} sec'.format(time_in_nanos / 1_000_000_000))
```

    0.08 sec


結果の転送に大部分の時間がかかっているようです。

検索件数を確認します。


```python
len(results['hits']['hits'])
```

    17006

他の結果と合っています。

2回目


```python
es = Elasticsearch(["http://localhost:9200"])
index_name = 'wiki_jp'
query = {"query" : {"match_phrase" : {"body" : "日本語"}}, "profile": "true"}
start_time = time.perf_counter()
results = es.search(index=index_name, body=query, size=20000)
print("{:.2f} sec".format(time.perf_counter() - start_time))
es.close()
```

    3.99 sec



```python
time_in_nanos = results['profile']['shards'][0]['searches'][0]['query'][0]['time_in_nanos']
print('{:.2f} sec'.format(time_in_nanos / 1_000_000_000))
```

    0.12 sec


キャッシュが効いています。

取得結果のひとつを確認します。


```python
results['hits']['hits'][0]
```

    {'_id': 'mFpPd4ABdhv1sUEQXV1x',
     '_index': 'wiki_jp',
     '_score': 9.08582,
     '_source': {'body': '日本語教育（にほんごきょういく）とは、外国語としての日本語、第二言語としての日本語についての教育の総称である。概要.日本語教育とは通常、日本語を母語としない人（主に外国人）に対し、日本国内外で、日本語を指導することを指す。ただし、日本語を母語とする人を対象とする「国語教育」を「日本語教育」と表す場合 もある。日本国外での日本語教育は126カ国・7地域で行われており、学習者は約300万人である。日本国内での日本語教育は、大学等の高等教育機関や日本語教育機関（主に日本語学校）の他、地域の日本語教室などで行われており、学習者は、成人が約166,000人、児童生徒約28,000人 と報告されている。また、日本語教育全般を取り扱う研究分野を「日本語教育学」と呼び、教育学の一分野として位置づけられる。日本語教育の歴史.明治以降第二次世界大戦まで.日本国内.1880年代前半より朝鮮からの留学生が増え、慶應義塾や陸軍戸山学校が受け入れ、1892年には文部省が「外国人留学規定」を制定した。1895年の日清戦争での日本の勝利により、日本統治となった台湾で日本語教育が始まったほか、近代化の必要性を自覚した清国から日本への留学生が増え、日本語教育の需要が急速に高まりはじめた。1898年には成城学校が清国陸軍留学生に日本語教育を開始したほか、高楠順次郎が清国人留学生の日本語教育のために東京本郷に日華学堂を設立、翌1899年には嘉納治五郎が清国からの国費留学生の受け入れ校として亦楽書院（のちの宏文学院）を設立、そのほか、第一高等学校、学習院、実践女学校などでも清国人留学生への日本語教育が始まり、留学生の急増により1904年には明治大学、法政大学に日本語を速成教育する専門科が新設された。1905年に日露戦争で日本が勝利したことにより、日本への留学熱は最高潮となり、同年の留学生は清国人だけでも8000人にのぼったが、教育法には不備も多く、辞書や教科書等の整備も始まった。清国での日本語教育も盛んになり、1907年には京師法政学堂（現・北京大学）でも日文教育が始まった。また、1907-08年にはベトナムから留学生300名が来日した（東遊運動）。清国人留学生は母国で革命の気運が高まると帰国者が相次いだが、1911年の辛亥革命以降、新生中華民国からの留学生が続々と来日し、中国人の日本留学ブームは1937年の日中戦争直前まで続いた。台湾からの留学生は1901年頃から増え、1922年には2400名以上を数えた。1935年に外務省文化事業部により「国際学友会」が設立され、中国を除く世界各地からの留学生の受入れを担当した。第二次世界大戦が始まると欧米などからの留学生は事実上途絶えたが、1943・1944年度は、大東亜省の指示により東南アジア諸地域より「南方特別留学生」が招かれた。第二次世界大戦後以降.バブル期の1985-86年ごろには、就学生相手に一商売をもくろんだ「日本語学校」が乱立し、社会問題化した。日本国内における日本語教育.日本国内における日本語教育実施機関・施設等で学ぶ日本語学習者数は、2015年（平成27年）11月現在、19万人に達している。これら留学生の所属別では、法務省告示機関（いわゆる日本語学校）が最も多く約7万人、大学等機関が約5万人、国際交流協会が約3万人、地方公共団体および教育委員会がそれぞれ約1万人となっている。また、在住外国人のために各地の地方自治体やNPOがボランティア教師による日本語講座や日本語教室を開催している。対象者は中国残留日本人孤児帰国者や日系人、国際結婚により来住した外国人女性などが多い。在住外国人増加につれ、同伴されてきたり日本で生まれたりした、日本語が母語ではない子どもたちが地域の小・中学校等に在籍することが増えてきている。また、海外で生まれ育ったために日本語指導が必要な日本人児童生徒も存在している。日本語指導が必要な外国人児童生徒は平成26年度で約29,000人、同じく日本人児童生徒は約8,000人にのぼり、うち約 75%が小学校に在籍していると報告されている。このような児童が多数在籍する学校には日本語指導を行う教員を加配する措置がある。少数在籍の学校では、ボランティア指導員が派遣される場合もある一方、全く日本語指導を受けられない場合もある。母語が確立していないこと、教科学習を行わなければならないことなど、子どもたちへの日本語教育は留学生や成人への日本語教育と異なる条件があるため、「年少者日本語教育」と呼ばれることもある。日本国外における日本語教育.世界全体の日本語学習者数は2015年（平成27年）現在で約365万人である。日本語は中国語と並んでアジアの主要な言語であり、日本国外の主要な大学には日本語学科が設置され、日本語を第二外国語として教える学校も多い。さらにオーストラリア、ベトナム、フランスなどでは初等・中等教育でも日本語教育が行われており、学習者数は格段に増加する。韓国の高校では日本語が第二外国語の一つになっており、中国語の次に履修率も高く、大学などでの履修者も含めると世界最大の日本語学習国であったが、2011年の中等教育の教育課程改定において第二外国語が必修科目から外されたこと、少子化が加速していることから、日本語学習者は漸減し続けている。中国の日本語教育は大学が中心だが、人口が多いので日本語学習者総数は95万人に達し、最大の日本語学習国となっている。日本語教育文法.日本語教育文法は、日本の公立学校で教わる学校文法とは、その内容を異にする。下記はその一例である。教授法.学習者にとって日本語は外国語（あるいは第二言語）であるので、指導に際しては外国語教授法が用いられる。言語学や学習理論、変形文法、認知学習理論、第二言語習得理論など様々な理論に基づいている。',
      'title': '日本語教育'},
     '_type': '_doc'}



次に、取得するフィールドをtitleのみに限定してみます。


```python
es = Elasticsearch(["http://localhost:9200"])
index_name = 'wiki_jp'
query = {"query" : {"match_phrase" : {"body" : "日本語"}}, "_source" : ["title"], "profile": "true"}
start_time = time.perf_counter()
results = es.search(index=index_name, body=query, size=20000)
print("{:.2f} sec".format(time.perf_counter() - start_time))
es.close()
time_in_nanos = results['profile']['shards'][0]['searches'][0]['query'][0]['time_in_nanos']
print('{:.2f} sec'.format(time_in_nanos / 1_000_000_000))
results['hits']['hits'][0]
```

    2.95 sec
    0.02 sec

    {'_id': 'mFpPd4ABdhv1sUEQXV1x',
     '_index': 'wiki_jp',
     '_score': 9.08582,
     '_source': {'title': '日本語教育'},
     '_type': '_doc'}


テキストの生成／転送が少ない分だけ時間が短くなっています。

本当にtitleだけを取得しているのか確認してみましょう。

```python
results['hits']['hits'][0]
```

    {'_id': 'mFpPd4ABdhv1sUEQXV1x',
     '_index': 'wiki_jp',
     '_score': 9.08582,
     '_source': {'title': '日本語教育'},
     '_type': '_doc'}



取得しているのはタイトルだけですね。

フィールドを取得しないで検索してみます。


```python
es = Elasticsearch(["http://localhost:9200"])
index_name = 'wiki_jp'
query = {"query" : {"match_phrase" : {"body" : "日本語"}}, "_source" : [""], "profile": "true"}
start_time = time.perf_counter()
results = es.search(index=index_name, body=query, size=20000)
print("{:.2f} sec".format(time.perf_counter() - start_time))
es.close()
time_in_nanos = results['profile']['shards'][0]['searches'][0]['query'][0]['time_in_nanos']
print('{:.2f} sec'.format(time_in_nanos / 1_000_000_000))
results['hits']['hits'][0]
```

    2.91 sec
    0.02 sec

    {'_id': 'mFpPd4ABdhv1sUEQXV1x',
     '_index': 'wiki_jp',
     '_score': 9.08582,
     '_source': {},
     '_type': '_doc'}



タイトルだけでも十分短いためか、ほぼ変わりません。

## curlで検索

ElasticSearchはREST APIとしてもアクセスできるので、curlを使って検索してみましょう。


```shell
%%time
!curl -XGET 'http://localhost:9200/wiki_jp/_search?pretty' -H 'Content-Type: application/json' -d '{"query": {"match_phrase": { "body": "日本語"}}, "size":20000}' > result
```

      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100  166M  100  166M  100    65  37.2M     14  0:00:04  0:00:04 --:--:-- 39.0M
    CPU times: user 68 ms, sys: 30.1 ms, total: 98.1 ms
    Wall time: 4.61 s


すでにキャッシュが効いているようです。PythonのAPIとほぼ同じです。

ちゃんと検索できているか内容を確認します。

```python
import json
with open('result') as fin:
    results = json.load(fin)
print(len(results['hits']['hits']))
results['hits']['hits'][0]
```

    17006

    {'_id': 'mFpPd4ABdhv1sUEQXV1x',
     '_index': 'wiki_jp',
     '_score': 9.08582,
     '_source': {'body': '日本語教育（にほんごきょういく）とは、外国語としての日本語、第二言語としての日本語についての教育の総称である。概要.日本語教育とは通常、日本語を母語としない人（主に外国人）に対し、日本国内外で、日本語を指導することを指す。ただし、日本語を母語とする人を対象とする「国語教育」を「日本語教育」と表す場合 もある。日本国外での日本語教育は126カ国・7地域で行われており、学習者は約300万人である。日本国内での日本語教育は、大学等の高等教育機関や日本語教育機関（主に日本語学校）の他、地域の日本語教室などで行われており、学習者は、成人が約166,000人、児童生徒約28,000人 と報告されている。また、日本語教育全般を取り扱う研究分野を「日本語教育学」と呼び、教育学の一分野として位置づけられる。日本語教育の歴史.明治以降第二次世界大戦まで.日本国内.1880年代前半より朝鮮からの留学生が増え、慶應義塾や陸軍戸山学校が受け入れ、1892年には文部省が「外国人留学規定」を制定した。1895年の日清戦争での日本の勝利により、日本統治となった台湾で日本語教育が始まったほか、近代化の必要性を自覚した清国から日本への留学生が増え、日本語教育の需要が急速に高まりはじめた。1898年には成城学校が清国陸軍留学生に日本語教育を開始したほか、高楠順次郎が清国人留学生の日本語教育のために東京本郷に日華学堂を設立、翌1899年には嘉納治五郎が清国からの国費留学生の受け入れ校として亦楽書院（のちの宏文学院）を設立、そのほか、第一高等学校、学習院、実践女学校などでも清国人留学生への日本語教育が始まり、留学生の急増により1904年には明治大学、法政大学に日本語を速成教育する専門科が新設された。1905年に日露戦争で日本が勝利したことにより、日本への留学熱は最高潮となり、同年の留学生は清国人だけでも8000人にのぼったが、教育法には不備も多く、辞書や教科書等の整備も始まった。清国での日本語教育も盛んになり、1907年には京師法政学堂（現・北京大学）でも日文教育が始まった。また、1907-08年にはベトナムから留学生300名が来日した（東遊運動）。清国人留学生は母国で革命の気運が高まると帰国者が相次いだが、1911年の辛亥革命以降、新生中華民国からの留学生が続々と来日し、中国人の日本留学ブームは1937年の日中戦争直前まで続いた。台湾からの留学生は1901年頃から増え、1922年には2400名以上を数えた。1935年に外務省文化事業部により「国際学友会」が設立され、中国を除く世界各地からの留学生の受入れを担当した。第二次世界大戦が始まると欧米などからの留学生は事実上途絶えたが、1943・1944年度は、大東亜省の指示により東南アジア諸地域より「南方特別留学生」が招かれた。第二次世界大戦後以降.バブル期の1985-86年ごろには、就学生相手に一商売をもくろんだ「日本語学校」が乱立し、社会問題化した。日本国内における日本語教育.日本国内における日本語教育実施機関・施設等で学ぶ日本語学習者数は、2015年（平成27年）11月現在、19万人に達している。これら留学生の所属別では、法務省告示機関（いわゆる日本語学校）が最も多く約7万人、大学等機関が約5万人、国際交流協会が約3万人、地方公共団体および教育委員会がそれぞれ約1万人となっている。また、在住外国人のために各地の地方自治体やNPOがボランティア教師による日本語講座や日本語教室を開催している。対象者は中国残留日本人孤児帰国者や日系人、国際結婚により来住した外国人女性などが多い。在住外国人増加につれ、同伴されてきたり日本で生まれたりした、日本語が母語ではない子どもたちが地域の小・中学校等に在籍することが増えてきている。また、海外で生まれ育ったために日本語指導が必要な日本人児童生徒も存在している。日本語指導が必要な外国人児童生徒は平成26年度で約29,000人、同じく日本人児童生徒は約8,000人にのぼり、うち約 75%が小学校に在籍していると報告されている。このような児童が多数在籍する学校には日本語指導を行う教員を加配する措置がある。少数在籍の学校では、ボランティア指導員が派遣される場合もある一方、全く日本語指導を受けられない場合もある。母語が確立していないこと、教科学習を行わなければならないことなど、子どもたちへの日本語教育は留学生や成人への日本語教育と異なる条件があるため、「年少者日本語教育」と呼ばれることもある。日本国外における日本語教育.世界全体の日本語学習者数は2015年（平成27年）現在で約365万人である。日本語は中国語と並んでアジアの主要な言語であり、日本国外の主要な大学には日本語学科が設置され、日本語を第二外国語として教える学校も多い。さらにオーストラリア、ベトナム、フランスなどでは初等・中等教育でも日本語教育が行われており、学習者数は格段に増加する。韓国の高校では日本語が第二外国語の一つになっており、中国語の次に履修率も高く、大学などでの履修者も含めると世界最大の日本語学習国であったが、2011年の中等教育の教育課程改定において第二外国語が必修科目から外されたこと、少子化が加速していることから、日本語学習者は漸減し続けている。中国の日本語教育は大学が中心だが、人口が多いので日本語学習者総数は95万人に達し、最大の日本語学習国となっている。日本語教育文法.日本語教育文法は、日本の公立学校で教わる学校文法とは、その内容を異にする。下記はその一例である。教授法.学習者にとって日本語は外国語（あるいは第二言語）であるので、指導に際しては外国語教授法が用いられる。言語学や学習理論、変形文法、認知学習理論、第二言語習得理論など様々な理論に基づいている。',
      'title': '日本語教育'},
     '_type': '_doc'}


結果をファイルに書き込む時間を除いて測定してみます。


```python
%%time
!curl -XGET 'http://localhost:9200/wiki_jp/_search' -H 'Content-Type: application/json' -d '{"query": {"match_phrase": { "body": "日本語"}}, "size":20000}'  > /dev/null
```

      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100  164M  100  164M  100    65  65.4M     25  0:00:02  0:00:02 --:--:-- 65.4M
    CPU times: user 58 ms, sys: 60.1 ms, total: 118 ms
    Wall time: 2.68 s


ファイルの書き込み時間は3分の1程度のようです。

出力をtitleだけにしてみます。


```shell
%%time
!curl -XGET 'http://localhost:9200/wiki_jp/_search?pretty' -H 'Content-Type: application/json' -d '{"query": {"match_phrase": { "body": "日本語"}}, "size":20000, "_source":"title"}' > result
```

      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 3611k  100 3611k  100    84   624k     14  0:00:06  0:00:05  0:00:01 1009k
    CPU times: user 66 ms, sys: 31.1 ms, total: 97.1 ms
    Wall time: 5.99 s


結果を確認します。


```python
import json
with open('result') as fin:
    results = json.load(fin)
print(len(results['hits']['hits']))
results['hits']['hits'][0]
```

    17006

    {'_id': 'mFpPd4ABdhv1sUEQXV1x',
     '_index': 'wiki_jp',
     '_score': 9.08582,
     '_source': {'title': '日本語教育'},
     '_type': '_doc'}


ファイル出力を除いてみます。

```shell
%%time
!curl -XGET 'http://localhost:9200/wiki_jp/_search?pretty' -H 'Content-Type: application/json' -d '{"query": {"match_phrase": { "body": "日本語"}}, "size":20000, "_source":"title"}' > /dev/null
```

      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 3611k  100 3611k  100    84   837k     19  0:00:04  0:00:04 --:--:--  837k
    CPU times: user 59.5 ms, sys: 82.1 ms, total: 142 ms
    Wall time: 4.47 s

出力量が少ないので、書き込み時間も少ないようです。

最後にコンテンツを取得しない検索

```shell
%%time
!curl -XGET 'http://localhost:9200/wiki_jp/_search?pretty' -H 'Content-Type: application/json' -d '{"query": {"match_phrase": { "body": "日本語"}}, "size":20000, "_source":""}' > result
```

      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 2765k  100 2765k  100    79   812k     23  0:00:03  0:00:03 --:--:--  812k
    CPU times: user 38.8 ms, sys: 33 ms, total: 71.8 ms
    Wall time: 3.61 s


```python
import json
with open('result') as fin:
    results = json.load(fin)
print(len(results['hits']['hits']))
results['hits']['hits'][0]
```

    17006

    {'_id': 'mFpPd4ABdhv1sUEQXV1x',
     '_index': 'wiki_jp',
     '_score': 9.08582,
     '_source': {},
     '_type': '_doc'}




```shell
%%time
!curl -XGET 'http://localhost:9200/wiki_jp/_search?pretty' -H 'Content-Type: application/json' -d '{"query": {"match_phrase": { "body": "日本語"}}, "size":20000, "_source":""}' > /dev/null
```

      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 2765k  100 2765k  100    79   945k     26  0:00:03  0:00:02  0:00:01  944k
    CPU times: user 39.1 ms, sys: 24.1 ms, total: 63.2 ms
    Wall time: 3.07 s


テキストを生成／転送しない分だけ時間が短くなっています。
