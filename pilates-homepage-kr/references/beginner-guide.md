# 초보자 가이드

## 용어 풀이 (처음 나올 때 이렇게 설명)

| 용어 | 쉬운 설명 |
|---|---|
| Next.js | 홈페이지를 만드는 도구 상자 |
| 로컬 / localhost:3000 | 내 컴퓨터에서만 보이는 미리보기 화면 |
| Supabase | 신청서·주문·회원 정보를 저장하는 온라인 창고 |
| RLS | 창고의 자물쇠 규칙. 누가 무엇을 볼 수 있는지 정함 |
| 환경변수 / .env.local | 비밀번호 같은 열쇠를 따로 넣어두는 금고 파일. 인터넷에 올리면 안 됨 |
| GitHub | 코드를 보관하는 온라인 보관함 |
| Vercel | 코드를 인터넷 주소로 띄워 주는 곳 |
| 배포 | 내 컴퓨터 미리보기를 인터넷에 공개하는 것 |
| PG / 토스페이먼츠 | 카드 결제를 대신 받아 주는 회사 |
| 테스트 키 / 라이브 키 | 연습용 열쇠 / 진짜 돈을 받는 열쇠 |
| DNS | 도메인 주소와 실제 서버를 이어주는 전화번호부 |
| 플레이라이트 | Claude가 브라우저를 대신 클릭해 주는 도구 |
| 스킬 | Claude에게 주는 작업 설명서. 이 파일이 바로 스킬 |

## 자주 나는 오류와 대처

| 증상 | 원인 | 대처 |
|---|---|---|
| `npm: command not found` | Node.js 미설치 | nodejs.org에서 LTS 버전 설치 후 터미널 다시 열기 |
| 화면이 하얗게 나오고 `Invalid supabaseUrl` | `.env.local` 값 누락·오타 | 값 다시 붙여넣고 `npm run dev` 재시작 |
| 신청서 저장 시 `new row violates row-level security` | RLS 정책 없음, 또는 동의 체크 안 함 | `database.md`의 정책 SQL 실행 여부 확인 |
| 구글 로그인 후 `redirect_uri_mismatch` | 구글 콘솔의 리디렉션 주소 오타 | `https://<프로젝트ID>.supabase.co/auth/v1/callback` 정확히 입력 |
| 배포 후 로그인하면 localhost로 이동 | Supabase Site URL이 localhost | URL Configuration에 배포 주소 추가 |
| 결제창이 안 뜨고 키 오류 | 결제위젯 키(`gck`)가 아닌 다른 키 사용 | `payments-toss.md`의 키 종류 확인 |
| 결제 후 "금액이 달라요" | 주문 금액과 결제 금액 불일치 | 주문 생성 API가 DB 가격을 쓰는지 확인 |
| Vercel에서는 안 되고 내 컴퓨터에서는 됨 | Vercel 환경변수 누락 | Settings → Environment Variables 확인 후 Redeploy |
| 도메인 연결이 계속 Invalid | DNS 전파 대기 또는 레코드 오타 | 1시간 기다린 뒤 가비아 DNS 값과 Vercel 화면 값을 한 글자씩 비교 |
| 챗봇이 아무 대답도 안 함 | `AI_GATEWAY_API_KEY` 누락 | 로컬과 Vercel 양쪽에 키 확인 |

## 원장님께 자주 하는 말
- "오류는 정상이에요. 하나씩 고치면 돼요."
- "이 단계는 원장님이 직접 로그인만 해 주시면 나머지는 제가 할게요."
- "결제 버튼이 나오는 단계부터는 원장님이 직접 눌러 주세요."
