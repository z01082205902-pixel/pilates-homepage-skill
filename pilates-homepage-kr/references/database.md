# Supabase 설정

Supabase = 신청서·주문·회원 정보를 저장하는 온라인 창고. 무료 플랜으로 충분하다.

## 공통 준비
1. https://supabase.com 가입 (사용자가 직접 로그인)
2. New project 생성. 지역은 **Northeast Asia (Seoul)**. 데이터베이스 비밀번호는 사용자가 정하고 따로 메모해 두게 한다.
3. Project Settings → API에서 값 두 개를 `.env.local`에 넣는다.
```
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
```
4. 패키지: `npm install @supabase/supabase-js @supabase/ssr`
5. `lib/supabase/client.ts`(브라우저용), `lib/supabase/server.ts`(서버용)를 만든다.
6. 아래 SQL을 Supabase **SQL Editor**에 붙여넣고 실행한다. (Claude가 브라우저로 대신 붙여넣어도 된다)

> RLS(Row Level Security) = "누가 어떤 줄을 보고 쓸 수 있는지" 정하는 보안 규칙. 절대 끄지 않는다.

---

## B. 홍보형: 신청서 테이블 하나

```sql
create table inquiries (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  phone text not null,
  preferred_time text,
  goal text,
  privacy_agreed boolean not null default false,
  created_at timestamptz not null default now()
);

alter table inquiries enable row level security;

-- 방문자는 신청서를 '넣기'만 할 수 있다 (남의 신청서는 못 본다)
create policy "anyone can submit"
  on inquiries for insert
  to anon, authenticated
  with check (privacy_agreed = true);
```
- 조회 정책은 만들지 않는다. 원장님은 Supabase 대시보드 Table Editor에서 본다.
- 신청 알림이 필요하면 나중에 Supabase Database Webhook이나 Zapier로 카톡·메일 알림을 붙일 수 있다고 한 줄만 안내한다.

---

## A. 결제형: 상품, 주문, 주차별 영상

```sql
create table products (
  id uuid primary key default gen_random_uuid(),
  slug text unique not null,
  name text not null,
  price integer not null check (price > 0),
  weeks integer not null,
  summary text,
  is_active boolean not null default true,
  created_at timestamptz not null default now()
);

create table orders (
  id uuid primary key default gen_random_uuid(),
  order_id text unique not null,          -- 토스에 보내는 주문번호
  user_id uuid not null references auth.users(id),
  product_id uuid not null references products(id),
  amount integer not null,
  status text not null default 'pending' check (status in ('pending','paid','canceled','refunded')),
  payment_key text,
  paid_at timestamptz,
  created_at timestamptz not null default now()
);

create table program_weeks (
  id uuid primary key default gen_random_uuid(),
  product_id uuid not null references products(id) on delete cascade,
  week integer not null,
  title text not null,
  video_url text,          -- 유튜브 '일부 공개' 링크
  homework text,
  unique (product_id, week)
);

alter table products enable row level security;
alter table orders enable row level security;
alter table program_weeks enable row level security;

-- 상품은 누구나 볼 수 있다
create policy "products are public" on products for select using (is_active = true);

-- 내 주문만 볼 수 있다 (주문 생성·결제 완료 처리는 서버에서 service role 키로)
create policy "own orders" on orders for select to authenticated using (user_id = auth.uid());

-- 결제 완료한 상품의 영상만 볼 수 있다
create policy "paid users see weeks" on program_weeks for select to authenticated
  using (exists (
    select 1 from orders o
    where o.product_id = program_weeks.product_id
      and o.user_id = auth.uid()
      and o.status = 'paid'
  ));

insert into products (slug, name, price, weeks, summary)
values ('online-pt-4w', '온라인 PT 4주', 180000, 4, '집에서 하루 20분, 원장님 피드백과 함께하는 4주 자세 관리');
```

- 서버 전용 키 `SUPABASE_SERVICE_ROLE_KEY`는 `.env.local`에만 넣고, 서버 코드(`app/api/...`)에서만 쓴다. 브라우저 코드에 절대 넣지 않는다.
- 관리자 화면(`/admin`)의 주문 조회와 영상 등록은 서버에서 `ADMIN_EMAILS` 확인 후 service role 키로 처리한다.

## A. 구글 로그인
1. https://console.cloud.google.com 에서 프로젝트 생성 (사용자 로그인)
2. APIs & Services → OAuth consent screen: 앱 이름(센터 이름), 지원 이메일 입력, External 선택
3. Credentials → Create OAuth client ID → Web application
   - Authorized redirect URI: `https://<프로젝트ID>.supabase.co/auth/v1/callback`
4. 받은 Client ID와 Client Secret을 Supabase → Authentication → Providers → Google에 넣고 켠다.
5. Supabase → Authentication → URL Configuration
   - Site URL: 처음엔 `http://localhost:3000`, 배포 후 Vercel 주소(또는 도메인)로 바꾼다
   - Redirect URLs에 `http://localhost:3000/**`와 배포 주소 `/**`를 추가
6. Next.js에 `app/auth/callback/route.ts`를 만들어 로그인 후 돌아오게 한다.

> 카카오 로그인은 카카오 개발자 사이트 검수에 며칠 걸린다. 구글로 먼저 완성하고, 원하는 사람만 나중에 추가한다.
