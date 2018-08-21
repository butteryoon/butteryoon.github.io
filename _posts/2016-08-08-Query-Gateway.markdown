---
layout: post
title:  "Query Gateway"
desc: "Query Build and Gateway"
---

**Query Builder and Gateway for Oracle DB**

**Requirement**

- 로그인 사용자는 쿼리 빌더에 접근 권한이 있어야 한다
- 사용자가 테이블 스키마 정보를 얻을 수 있어야 한다. 
- 테이블에서 원하는 컬럼을 선택할 수 있어야 한다. 
- JOIN, WHERE 조건을 정의할 수 있어야 한다. 
- 작성한 템플릿을 저장할 수 있어야 한다. 
  - 템플릿은 사용자별로 선택 가능.
- 작성한 쿼리의 정합여부를 확인할 수 있어야 한다. 
  - 쿼리를 실행하여 샘플 데이타 결과 보여줌 (기간, Row개수, etc)
- 쿼리 템플릿 제공 
- 사용자 쿼리 저장 및 불러오기 기능

**Query Buiilder Open Source** 

- [Knex](http://knexjs.org/)
- [RedQueryBuilder](http://redquerybuilder.appspot.com/)

