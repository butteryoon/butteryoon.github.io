# _reports

Perplexity 배치로 생성되는 RAG & AI 에이전트 주간 보고서 저장 폴더.

- 파일명 형식: `YYYYMMDD_report.md` (해당 주 일요일 날짜)
- 매주 일요일 18:00 KST에 클라우드 루틴이 이 폴더의 최신 보고서를 읽어
  `_posts/`에 요약 글을 발행한다. 보고서가 없는 주는 건너뛴다.
- 이 폴더는 Jekyll 빌드에 포함되지 않는다 (underscore 디렉토리).
