# GitHub 백업, Vercel 배포, 가비아 도메인

- GitHub = 코드를 저장해 두는 온라인 보관함 (백업)
- Vercel = 그 코드를 받아 인터넷 주소로 띄워 주는 곳 (배포)
- 가비아 = 도메인(www.내센터.com 같은 주소)을 사는 곳

## 1. 배포 전 비밀키 점검 (필수)
```bash
git ls-files | grep -E "\.env" && echo "위험: .env 파일이 올라가려 함" || echo "OK"
grep -E "^\.env" .gitignore || echo "주의: .gitignore에 .env 없음"
```
코드 안에 `test_gsk_`, `live_gsk_`, `service_role`, `AI_GATEWAY` 값이 직접 적혀 있지 않은지도 검색한다.
이미 올라갔다면: 해당 키를 **새로 발급**하고 예전 키는 폐기한다. 기록만 지우는 걸로 끝내지 않는다.

## 2. GitHub 비공개 저장소
- `gh`가 로그인돼 있으면: `gh repo create <이름> --private --source=. --push`
- 아니면 github.com에서 New repository → **Private** → 안내대로 push
- 커밋 이메일은 사용자가 알려준 이메일로 `git config user.email` 설정

## 3. Vercel 연결 (자동 배포)
1. vercel.com 에 GitHub 계정으로 가입·로그인 (사용자가 직접)
2. Add New → Project → 방금 만든 저장소 Import
3. Environment Variables에 `.env.local`의 값을 모두 넣는다
   - 공통: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `AI_GATEWAY_API_KEY`
   - A만: `SUPABASE_SERVICE_ROLE_KEY`, `NEXT_PUBLIC_TOSS_CLIENT_KEY`, `TOSS_SECRET_KEY`, `ADMIN_EMAILS`
4. Deploy → `https://<이름>.vercel.app` 주소가 생긴다
5. 이후로는 GitHub에 push할 때마다 자동으로 다시 배포된다
6. [A] Supabase → Authentication → URL Configuration의 Site URL과 Redirect URLs에 Vercel 주소를 추가한다 (안 하면 로그인 후 localhost로 튕김)

## 4. 가비아 도메인 연결 (도전 과제)
1. 사용자가 gabia.com 가입 후 원하는 도메인을 검색해 **직접 결제**한다. (.com, .kr, .co.kr 등. 저렴한 도메인도 운영에 문제없다)
2. Vercel → 프로젝트 → Settings → Domains → 산 도메인 입력 (예: `barunsum.com`과 `www.barunsum.com`)
3. Vercel이 보여주는 DNS 값을 확인한다. 보통:
   - 루트 도메인(`@`): **A 레코드** → Vercel이 알려준 IP
   - `www`: **CNAME** → Vercel이 알려준 주소 (예: `cname.vercel-dns.com` 형태)
   - Vercel 화면에 표시된 값이 정답이다. 이 문서의 예시보다 화면 값을 따른다.
4. 가비아 → My가비아 → 도메인 관리 → DNS 정보 → DNS 설정에서 위 레코드를 추가하고 저장
5. 연결까지 몇 분~몇 시간 걸린다. Vercel Domains 화면에서 "Valid Configuration"이 뜨면 완료. HTTPS 인증서는 Vercel이 자동으로 붙인다.
6. [A] Supabase URL Configuration과 Google OAuth 설정에 새 도메인을 추가한다. 토스 심사 신청서에도 이 도메인을 쓴다.

브라우저 자동화로 진행할 때: 가비아·Vercel 로그인은 사용자가, 결제는 사용자가, DNS 입력과 저장은 Claude가 해도 된다. 저장 전 입력값을 사용자에게 한 번 보여준다.
