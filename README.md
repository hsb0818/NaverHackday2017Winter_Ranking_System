# NAVER CAMPUS HACKDAY 2017 Winter - comment

### - 댓글 증감수 기반 콘텐츠 랭킹 시스템 개발

## 주제 선정 배경
##### - 수많은 컨텐츠가 이미 생성 되어있고 새로 생성이 되는데, 컨텐츠에 작성된 댓글 최신 목록을 조회하고 특정 기간 동안 댓글의 증감을 표시하고 랭킹순으로 조회할 수 있는 페이지를 개발해 보자!

## 목표
##### - 특정 기간 동안(최소 1분 간격)의 모든 컨텐츠들 간 댓글 증감수에 따른 랭킹 및 최신 댓글 N개 조회 가능한 API 개발

## 개발환경
 - <h3>Ubuntu 16.04 LTS</h3>
 - <h3>MySQL 5.7.17</h3>
 - <h3>Spring Framework</h3>
 - <h3>JDK 1.8</h3>

### * DB Server : 멘토님이 제공해 주심.


## DataBase Table Structure

![database structure](https://raw.githubusercontent.com/hsb0818/NaverHackday2017Winter_Ranking_System/master/src/db0.jpg)
#### Database : comment
#### Tables : cmt_cnts, hsb\_cache, hsb\_rank, hsb\_rankspan
---
### cmt_cnts
![cmt_cnts](https://raw.githubusercontent.com/hsb0818/NaverHackday2017Winter_Ranking_System/master/src/cmt_cnts.png)
### - 실시간으로 초당 500개의 댓글이 무작위 컨텐츠에 무작위로 INSERT된다.
### hsb_rank
![hsb_rank](https://raw.githubusercontent.com/hsb0818/NaverHackday2017Winter_Ranking_System/master/src/hsb_rank.png)

- 특정 날짜의 Service의 Content에 대한 댓글 증감수를 분 단위로 모두 기록해 Rank를 매기는 곳이다.
- hsb\_cache에 데이터가 추가/수정될 때마다, 적용된 Trigger가 실행되어 이곳에 해당 댓글의 Service ID와 Contents ID, 그리고 분 단위의 날짜 값이 모두 일치하는 Row를 찾아 댓글 증감을 표시하게 된다.  
- PK로 인해 Indexing이 되어 있는 Column들이 있기 때문에, 그에 해당하는 Column에 대한 ORDER BY문을 통한 READ에도 성능이 좋다.
	- WRITE에 대해 수행함에 있어 신경 쓸 만 한 부하가 없었다.   

### hsb_rankspan
![hsb_rankspan](https://raw.githubusercontent.com/hsb0818/NaverHackday2017Winter_Ranking_System/master/src/hsb_rankspan.png)

- hsb_rank로부터 분 단위로 저장된 랭킹들로부터 Top 100을 선정하여 저장하는 테이블이다. 최종적으로 이 테이블을 통해 API 기능을 제공하게 되며, 유저는 날짜, Service ID, Contents ID를 통해 데이터를 제공받을 수 있다.


### hsb_cache
![hsb_cache](https://raw.githubusercontent.com/hsb0818/NaverHackday2017Winter_Ranking_System/master/src/hsb_cache.png)

- 서버가 실행된 후로부터는 cmt_cnts에 추가되는 댓글들이 이 테이블에도 추가된다.
- Delete요청마다 삭제하면 느리고, 복구를 해야 할 수 있으므로 del_yn column을 만들어 UPDATE 요청으로 mark만 해 놓는 방식을 이용한다.
- MySQL Procedure 코드를 작성해 INSERT / UPDATE에 Trigger를 적용해 놓았다.
	- INSERT 시 : hsb\_rank에 작성된 날짜(reg_ymdt에 저장됨)를 1711240312와 같이 분 단위의 Integer로 변환하여 저장해 주는데, hsb\_rank에 이미 존재한다면 댓글 증감 수치만 1 증가시킨다.
	- UPDATE 시 : INSERT와 같은 처리를 하지만, 수정된 날짜에 대해 처리해야 하므로 mod_ymdt(수정된 날짜) 데이터를 이용하여 저장한다.
	- __UPSERT__를 이용하여 효율적으로 처리했다.

## Implementation
![total Structure](https://raw.githubusercontent.com/hsb0818/NaverHackday2017Winter_Ranking_System/master/src/total_structure.jpg)
### 1) 댓글?
- 1분에 3만 건의 댓글이 무작위로 INSERT / UPDATE 된다.
- 새로운 컨텐츠는 계속 새로 추가된다.

### 2) Cache를 만들어, 서버 실행 이후의 데이터를 저장.
- 형식은 cmt_cnts 테이블에 저장되는 데이터와 동일하다.
- 댓글 수가 아닌, 댓글 '증감'수 이므로 서버 실행 이전의 모든 댓글들을 분석할 필요는 없다.\
- cmt\_cnts 테이블은 모든 댓글에 대한 데이터가 저장되는 곳. hsb\_cache 테이블은 분석이 끝나면 삭제해도 상관 없는 테이블이다.

### 3) MySQL Trigger on Cache
- hsb\_cache 테이블에 MySQL Procedure 코드를 작성해 hsb\_rank 테이블에 대한 INSERT와 UPDATE(삭제도 UPDATE다) Trigger를 등록시켜 놓는다. 이로 인해 따로 Spring Server에서 Query 요청을 관리할 필요가 없어지고, 일련의 과정은 하나의 Transaction처럼 이루어진다.
- 이 과정을 통해 hsb\_cache에 INSERT / UPDATE 쿼리 요청 시 Trigger가 실행되어 자동으로 hsb\_rank에 랭킹이 갱신되며 저장된다.
- 즉 이 단계까지는 Database에 프로그래밍하는 것만으로 해결되었으므로 __Spring Server가 필요가 없었다.__  

### 4) Daemon by Cron
- Spring Server에서 1분 단위로 hsb\_rank 테이블로부터 Top 100을 선정하여 hsb\_rankspan 테이블에 저장해 준다.
- 이 작업은 Cron scheduling을 통해 Background에서 이루어진다.


#### 이렇게 API 개발이 완료되었다!

## 결과
이 구조로 구현하여 성능을 테스트 해 본 결과, 실제로 분당 3만 개의 데이터에 대한 처리 시간은 __10~15초__로 안정적이었고, 이 API에 대한 성능으로는 이미 hsb\_rankspan에 top100으로 정렬되어 저장되어 있는 곳에서 검색하는 것이므로 문제가 없었다.

## 해커톤 소감
이렇게 방대한 양의 데이터에 대한 처리를 실제로 해 본 적이 없어서 걱정을 많이 했었다. 그렇지만 멘토님의 가르침을 통해 실제로 이런 데이터들을 어떻게 처리하고 있는지에 대해 파악할 수 있었고, 실제로 그와 관련된 API를 개발해 보고 잘 작동하는 모습을 보니 정말 행복했다..,. 같은 팀 분들도 너무 착하고 좋았다.. 최고.
