# Match Phrase Prefix로 Path 검색



우선 Elasticsearch가 어떻게 [저장](elasticsearch.md)되고 [검색](elasticsearch-1.md)되는지는 앞 선 글에 더 자세하게 적어두었다



## Problem

팀에서 Elasticsearch로 구현된 부분에 이런 문의가 들어왔다,

> /documents/common 을 검색했는데, /documents/suzie/common도 검색됐어요!

코드를 보니 Match Query가 and option으로 설정되어 있었다.. 어떤 검색 쿼리를 써야 적당한 걸까?!\
(이미 저장되어있는 데이터이기 때문에, 인덱싱 방식을 바꾸는 것 말고 검색 방식을 최적화 하는 방안을 고민했다)



## As-is : Match Query

검색어도 Analyze 하는 쿼리 (term은 검색어 자체 검색)

* operator: or, and (default: or)
  * or: token 하나라도 매칭되면 return
  * **and: 모든 token이 있어야 return**

```
💡 아하 그럼 documents, common이 둘 다 들어있는 데이터를 검색해서, 순서가 엉킨 것도 검색했구나?
```



**⇒ 그럼 순서를 지키는 Match Query를 알아보자**

### Match Phrase Query

Match Query(and) + **순서가 일치해야 함** 따라서 일치하는 문장을 찾을 때 주로 사용

* slop: 단어 사이의 거리(기본값 0)
  * ex) slop: 1 -> 단어 사이의 거리가 1인 것도 검색

### Match Phrase Prefix Query

Match Phrase Query + 마지막 단어 접두사로 취급해 부분 일치 검색 허용 도대체 이게 무슨말이냐고...?! \
-> **마지막 단어 자동 완성 느낌이라고 생각하면 될 것 같다!**

* ex) "I am Su" -> "I am Suzie" 검색 가능



## Solution

기존 코드를 수정하는 것이기 때문에 검색 query만 바뀌길 원했다. Match Query(And)-> Match Phrase Prefix Query로 변경하면 path 의 순서가 꼬이는 것을 방지하고 사용자가 like 과 비슷하게 기대하던 검색을 할 수 있을 것이라고 생각했다.

그런데 es에서 확인하고 프로젝트에 적용하니, 이러한 에러를 만났다 ㅜㅜ

```
[match_phrase_prefix] query does not support [zero_terms_query]
```

stackoverflow 에서 찾은 이유는 es client 버전과 es 버전이 맞지 않는 이유라고 했다.(match\_phrase\_prefix가 zero\_terms\_query를 지원하는 건 es 7.10 이후부터 가능하다고 한다.)

그래서 대안으로 Multi Match Query에 필드를 하나만 넣게 설정하고, phrase\_prefix 옵션으로 설정해서 사용했다.

### Multi Match Query

하나의 쿼리로 여러 필드를 검색할 수 있게!(필드 별 가중치 줄 수 있음)

* best\_fields: (기본값) 검색어와 가장 잘 매칭되는 단일 필드를 기준으로 결과를 반환
* most\_fields: 검색어가 여러 필드에서 매칭될 때, 각 필드의 매칭 점수를 합산하여 결과를 반환
* cross\_fields: 검색어를 여러 필드를 조합하여 하나의 필드처럼 검색(합치는 느낌!)
* phrase: 검색어를 구문으로 간주하여, 구문 일치 검색을 수행
* **phrase\_prefix: 구문 일치 검색을 하되, 마지막 단어를 접두사로 취급하여 검색**



찾아보면서 Path Hierachy등 맞춤 옵션을 많이 찾았지만, 기존 사용하던 부분은 다른 검색에 전체적으로 사용되었던 모듈이기 때문에, 공통적으로 match\_phrase\_prefix로 변경하는 것이 더 낫겠다고 판단했다.



## Outro

그럼 like 검색을 Elasticsearch에서는 어떻게 하면 될까? like 검색을 하고 싶다면 field를 keyword로 저장하고 wildcard로 검색하는 방법이 있을 것이다. 하지만 이 방법은 Elasticsearch의 Indexing 장점을 활용하지 못하기 때문에 데이터가 많을 경우 지양하는 게 좋을 것이댜.

문장, 혹은 특수문자로 연결된 단어 검색이라면 match phrase prefix가 대안이 될 수 있다.&#x20;

* token이 모두 들어있는지 확인하고
* 그 token이 순차적으로(사이에 다른 토큰 없이) 나열되어있는지 확인하고
* 검색어의 마지막 단어는 접두어로 부분 일치 검색이 가능하기 때문에 사람이 검색하기에 그나마 자연스러운 검색을 지원할 수 있다.(앞에서 부터 검색한다는 가정하에 like 검색과 유사하다고 생각함)

하지만 상황에 따라 적절하지 않은 케이스도 많을 것이기 때문에, 세밀한 검색은 field 별로 analyzer 설정을 해야할 것이다.

그리고 단어 단위가 아니라 글자단위의 like 검색이 필요하다면 `ngram으로 인덱싱을 많이 하기 vs wildcard로 검색을 비효율적으로 하기`에서 Trade-off를 결정해야할 것 같다



### Reference

https://esbook.kimjmin.net/06-text-analysis\
https://findstar.pe.kr/2018/01/19/understanding-query-on-elasticsearch/ https://discuss.elastic.co/t/difference-between-analyzer-and-normalizer/205897/2 https://stackoverflow.com/questions/71839329/how-to-remove-zero-terms-query-in-match-phrase-prefix-in-elasticsearch-from-quer
