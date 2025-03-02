# Elasticsearch 저장 방법

뛰어난 검색 기능을 활용하게 위해 Elastic Search를 사용했지만, 제대로 이해한 채로 사용하지 않아 의도한대로 검색되지 않는다면 의미가 없을 것이다,,! ES가 어떻게 데이터를 저장하고, 검색하는지 알아보자!!

## Inverted Index

Elasticsearch는 저장하는 행위를 Indexing 한다고 하는데, 이건 저장하면서 (Inverted) Index를 생성하기 때문이다!

* 구조 : Text를 Tokenize한 Term에 대한 Document ID를 저장하는 구조
  * ex) "suzie" -> doc1, doc2, doc3 (suzie가 포함된 문서)

```
* 뭐가 달라??
기본적으로 RDBMS는 like 검색을 위해 데이터를 순차 검색 -> 속도 느림
ES는 데이터를 저장할 때 Inverted Index를 생성(색인) -> 빠른 검색 가능
```

아니 근데, 무슨 기준으로 text를 indexing 하는건데?!

## Analyzer

텍스트를 Indexing하는 과정에서 Analyzer가 텍스트 분석을 하게된다.

### ElasticSearch Analyzer Pipeline

![image](https://github.com/suzieep/TIL/assets/61377122/4502420f-f1b0-4ac2-8101-8f862f4a4942)

#### Char Filters

입력된 원본의 텍스트를 분석에 필요한 형태로 변환(전처리)

* **HTML Strip** : HTML -> 일반 텍스트
* **Mapping** : 지정 단어 치환
  * ex) 특수문자(+) -> 문자(\_plus\_) 치환
* **Pattern Replace** : 정규식(Regular Expression)으로 복잡한 경우 치환
  * camelCase -> 사이에 공백 삽입
* ...

#### Tokenizer

입력 데이터를 설정된 기준에 따라 검색어 토큰으로 분리하는 역할

* **standard**: 기호, 공백을 구분자로 분리
  * ex) -, \[], @, /
* **whitespace**: 스페이스, 탭, 그리고 줄바꿈 같은 공백만을 기준으로 텀을 분리
  * ex) 띄어쓰기, 탭, 줄바꿈
* **keyword**: 입력값 전체를 하나의 싱글 토큰으로 저장
* **letter**: 알파벳을 제외한 모든 공백, 숫자, 기호들
* **UAX URL Email**: 이메일 주소와 웹 URL 경로는 분리하지 않고 그대로 하나의 텀으로 저장
* **Pattern**: 분리할 패턴 지정
  * ex) 단일 기호('/'), 정규식(camelCase) 등으로 지정 가능
* **Path Hierachy**: 경로 데이터를 계층별로 저장해서 하위 디렉토리에 속한 도큐먼트들을 수준별로 검색 가능
  * ex) /hi/hello/bye -> /hi, /hi/hello, /hi/hello/bye
* ...

#### Token Filters

분리된 토큰들에 다시 필터를 적용해서 실제로 검색에 쓰이는 검색어들로 최종 변환하는 역할

* **keyword**: 입력값 전체를 하나의 싱글 토큰으로 저장
* **Lowercase**: lowercase로 변경해 저장(Uppercase)
* **Stop**: "in", "the" 같은 무의미한 단어를 제외할 수 있음
* **Synonym**: 유의어, 약어 등을 사전에 설정
* **Unique**: 중복되는 검색어 삭제
* **nGram**: 입력값을 미리 분리해서 저장
  * min\_gram (default: 1), max\_gram (default: 2)
  * ex) "suzie" (size 1\~2) => s/u/z/i/e, su/uz/zi/ie
* **edgeNGram**: nGram을 앞에서부터 검색할 때
  * 예: "suzie" (size 1\~3) => s, su, suz
* **Shingle**: nGram을 문장으로 단어 단위로 검색할 때
  * min\_shingle\_size (default: 2), max\_shingle\_size (default: 2)
  * ex) "I am Suzie" (size 2) => I am, am Suzie
* ...

**Stemmer**

어간 추출, 형태소 분석

* **Snowball**: 형태소 분석 알고리즘("s","ing" 등 제거)
* **Nori**: 한글 형태소 분석기



이제 이렇게 분석해서 저장한 데이터를 [어떻게 검색할지는 다음 글](elasticsearch-1.md)에서 확인 해 보자!



### Reference

https://esbook.kimjmin.net/06-text-analysis\
https://findstar.pe.kr/2018/01/19/understanding-query-on-elasticsearch/ https://discuss.elastic.co/t/difference-between-analyzer-and-normalizer/205897/2 https://stackoverflow.com/questions/71839329/how-to-remove-zero-terms-query-in-match-phrase-prefix-in-elasticsearch-from-quer
