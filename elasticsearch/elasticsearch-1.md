---
description: Elastic Search에 어떤 Query가 있고, 각각 어떤 검색을 제공하는지 알아보자
---

# Elasticsearch 검색 방법



Elastic Search에 어떤 Query가 있고, 각각 어떤 검색을 제공하는지 알아보자

그 전에, Elastic Search가 어떻게 데이터를 [저장](elasticsearch.md)하는지는 앞선 글에 정리해뒀다:-)&#x20;



## Term Query

ES는 저장할 때 들어온 Text를 Token으로 분리하고, 이를 Term에 대한 Document ID를 저장하는 구조라고 했었다. 그래서 Term Query는**말 그대로 색인된 Term이 일치하는 것을 찾는 Query**이다!!

```
ex) ES 저장 Text : '여러개의 물건들'
 -> 색인된 Term : '여러', '개', '물건', '물건들'
```

이 때 Term Query로 검색할 수 있는 Term은 '여러', '개', '물건', '물건들'만 되는 것이다!

* 따라서, keyword field 검색에 주로 사용(**text field에는 사용 지양**)
* analyzer는 안 거치지만 대상 필드에 normalizer가 설정되어 있으면 normalizer를 거친다

```
* ES에서 선언 가능한 문자열 타입
- text: Analyzer를 통해 Tokenize된 Term을 저장
- keyword: 입력된 문자열을 하나의 Token으로 저장(= text타입에 keyword analyzer 적용한 것과 같음)
    ㄴ 주로 정렬, 필터링에 사용
```

#### Normalizer

* Analyzer와 비슷하지만, token을 분리하지 않는 점이 다르다!!(Tokenizer X)
* CharFilter, TokenFilter 일부 가능
  * lowercase 가능, stemming 같은 전체 형태소 분석 불가능

#### Wildcard Query

* 주로 keyword field에 사용
* wildcard 문자: \*, ? 로 부분일치 검색 가능
  * ex) \*suzie\*, suz?e
* 전체 스캔이므로 주의해서 사용!!

## Match Query

ES Query 종류를 알아보고자 검색하다보면 아래 문장을 계속 마주치게 된다.

> Match query 는 Term query와 달리 Analyzer를 거쳐 검색된다 \
> 아! 그러니까 Match 쿼리는 **검색어도 Analyze해서 검색**하는거구나?!

* operator: or, and (default: or)
  * or: token 하나라도 매칭되면 return
  * and: 모든 token이 있어야 return
* zero\_terms\_query: none, all (default: none)
  * 쿼리 문자열이 비어 있거나 모든 토큰이 제거된 경우, 어떻게 처리할지 설정
  * none: 검색 결과가 없음
  * all: 모든 문서가 반환

### Match Phrase Query

Match Query(and) + **순서가 일치해야 함** 따라서 일치하는 문장을 찾을 때 주로 사용

* slop: 단어 사이의 거리(기본값 0)
  * ex) slop: 1 -> 단어 사이의 거리가 1인 것도 검색

### Match Phrase Prefix Query

Match Phrase Query + 마지막 단어 접두사로 취급해 부분 일치 검색 허용 도대체 이게 무슨말이냐고...?! \
-> **마지막 단어 자동 완성 느낌이라고 생각하면 될 것 같다!**

* ex) "I am Su" -> "I am Suzie" 검색 가능

### Multi Match Query

하나의 쿼리로 여러 필드를 검색할 수 있게!(필드 별 가중치 줄 수 있음)

* best\_fields: (기본값) 검색어와 가장 잘 매칭되는 단일 필드를 기준으로 결과를 반환
* most\_fields: 검색어가 여러 필드에서 매칭될 때, 각 필드의 매칭 점수를 합산하여 결과를 반환
* cross\_fields: 검색어를 여러 필드를 조합하여 하나의 필드처럼 검색(합치는 느낌!)
* phrase: 검색어를 구문으로 간주하여, 구문 일치 검색을 수행
* phrase\_prefix: 구문 일치 검색을 하되, 마지막 단어를 접두사로 취급하여 검색



### Reference

https://esbook.kimjmin.net/06-text-analysis\
https://findstar.pe.kr/2018/01/19/understanding-query-on-elasticsearch/ https://discuss.elastic.co/t/difference-between-analyzer-and-normalizer/205897/2 https://stackoverflow.com/questions/71839329/how-to-remove-zero-terms-query-in-match-phrase-prefix-in-elasticsearch-from-quer
