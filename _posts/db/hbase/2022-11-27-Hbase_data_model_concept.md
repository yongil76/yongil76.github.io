---
title:  "Hbase data model concept"
excerpt: "Hbase data model concept"

categories:
- DB
  
tags:
- hbase

toc: true

toc_sticky: true

toc_label: "hbase"

last_modified_at: 2022-11-27T07:37:00-05:00

---

## Hbase data model

- Hbase에서는, 테이블에 데이터가 row와 column을 가진 형태로 저장된다. 
- RDBMS(Oracle, Mysql)와 용어적으로 겹치는 부분이 존재해보지만, 관계형 데이터베이스와는 전혀 다르다. 
- Hbase는 다차원 형태의 Map으로서 테이블을 이해하는 것이 도움이 될 것이다.

~~~json
{
  "com.cnn.www": {
    contents: {
      t6: contents:html: "<html>..."
      t5: contents:html: "<html>..."
      t3: contents:html: "<html>..."
    }
    anchor: {
      t9: anchor:cnnsi.com = "CNN"
      t8: anchor:my.look.ca = "CNN.com"
    }
    people: {}
  }
  "com.example.www": {
    contents: {
      t5: contents:html: "<html>..."
    }
    anchor: {}
    people: {
      t5: people:author: "John Doe"
    }
  }
}
~~~


## Table
- Table은 다수의 Row들로 구성되어 있다.
![](/assets/images/corutine/hbase/HBase-Table-HBase-Architecture-Edureka.png)


## Row
- Hbase의 Row는 하나의 Row key와 다수의 값을 가진 컬럼들로 이뤄져있다.
- Row는 저장될 때 결정된 Row key에 의해서, 문자스럽게(alphabetically) 정렬된다.
- 정렬의 대상이 되는 Row key의 설계는 매우 중요하다.
- 정렬이 중요한 이유는 필요한 데이터들을 서로 근처에 둬서 효율적으로 데이터를 저장하는 것이 목적이다.
- 만약에 Domain의 주소들을 저장하는 상황이라고 생각하면,
  - org.apache.www, org.apache.mail, org.apache.jira가 있는 상황
  - Rowkey로 지정되면, 테이블 내에서 서로의 근처가 있기 때문에,
  - org라고 검색이 됐을 때, 위 세 개의 데이터들이 근접하기 때문에 빠른 속도로 결과를 얻을 수 있다.


## Column
- Hbase의 Column은 Column Family와 Column Qualifier로 구성되어 있다.
- Column을 유윌하게 구분하는 값은 {Column Family}:{Column Qualifier}
  - {Student}:{Name}


## Column Family
- Column, Value들에 대한 정보를 포함
- 각 Column Family는, 아래의 정보들을 포함할 수 있다.
  - 메모리에 캐시에 필요한 정보
  - 데이터들을 압축하는 방식
  - Rowkey를 인코딩하는 방식
- 테이블내의 모든 Row는 동일한 Column Family를 가질 수 있다.


## Column Qualifier
- Column Qualifier은 Column Family에 주어진 데이터의 Index를 제공하기 위한 목적
- {Column Qualifier}:{Column Family}
- Column Family는 테이블이 만들어질 때, 고정되지만, Column Qualifier은 각 Row마다 선택적으로 포함할 수 있음
  - RDBMS와 다르게, 선택적으로 Column을 사용할 수 있으므로 데이터 저장 측면에서 효율적


## Cell
- Cell은 Row, Column Family, Column Qualifier의 결합을 의미
- Value와 Value's Version을 의미하는 Timestamp를 포함


## Timestamp
- 값의 버전 관리를 위해서 사용된다
- 기본적으로, Timestamp는 Region Server에 데이터가 쓰여진 시간을 의미


## Reference
- [Apache hbase](https://hbase.apache.org/book.html#datamodel)