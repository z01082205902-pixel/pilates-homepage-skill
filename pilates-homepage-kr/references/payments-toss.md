# 토스페이먼츠 결제위젯 (A 결제형 전용)

토스페이먼츠 = 카드·간편결제를 받아 주는 결제 대행사(PG). 결제위젯을 쓰면 카드, 토스페이, 카카오페이 등이 한 화면에 나온다.

## 원장님께 먼저 알려줄 것
- 지금은 **테스트 키**라서 결제창은 진짜처럼 뜨지만 돈은 빠져나가지 않는다.
- 실제 돈을 받으려면: 사업자등록 → 통신판매업 신고 → 토스페이먼츠 가입·심사 → 라이브 키로 교체. 심사 기간은 보통 며칠 이상 걸린다.
- 심사 때 홈페이지에 **상품·가격, 이용약관, 환불규정, 개인정보처리방침, 하단 사업자 정보**가 보여야 한다 (`legal-pages.md`).

## 1. 설치와 키

```bash
npm install @tosspayments/tosspayments-sdk
```

`.env.local` (테스트용. 토스 개발자센터 문서에 공개된 결제위젯 테스트 키를 넣는다. 또는 가입 후 내 개발자센터 → API 키의 **결제위젯 연동 키** 테스트 값을 쓴다)
```
NEXT_PUBLIC_TOSS_CLIENT_KEY=test_gck_...
TOSS_SECRET_KEY=test_gsk_...
```
- 결제위젯은 `gck`/`gsk`로 시작하는 **결제위젯 연동 키**를 쓴다. `ck`/`sk`(API 개별 연동 키)와 섞으면 오류가 난다.
- 키 값이나 SDK 사용법이 바뀌었을 수 있으니 시작 전에 https://docs.tosspayments.com 의 결제위젯 가이드를 한 번 확인한다.

## 2. 흐름

```
[결제 페이지] 로그인 확인
   → 서버에 주문 생성 요청 (POST /api/orders) : 상품 가격을 DB에서 읽어 orders에 pending으로 저장, orderId 반환
   → 결제위젯 렌더링 → 사용자가 결제하기
   → 토스가 /checkout/success?paymentKey=...&orderId=...&amount=... 로 돌려보냄
[성공 페이지] 서버에 승인 요청 (POST /api/payments/confirm)
   → 서버: orders의 금액과 amount가 같은지 확인 → 토스 승인 API 호출 → 성공하면 status='paid'
   → "내 프로그램으로 가기"
```

## 3. 주문 생성 API — `app/api/orders/route.ts`
- 로그인한 사용자 확인 (없으면 401)
- 요청에서 받은 `slug`로 `products`를 조회해 **DB 가격**을 쓴다 (화면에서 보낸 금액은 쓰지 않음)
- `orderId`는 `PT-<날짜>-<랜덤 8자>`처럼 6~64자, 영문·숫자·`-`·`_`만
- service role 키로 `orders`에 `pending` 저장 후 `{ orderId, amount, orderName }` 반환

## 4. 결제 화면 — 클라이언트 컴포넌트

```tsx
"use client";
import { useEffect, useRef, useState } from "react";
import { loadTossPayments, type TossPaymentsWidgets } from "@tosspayments/tosspayments-sdk";

export function CheckoutWidget(props: {
  userId: string; orderId: string; orderName: string; amount: number;
  customerEmail?: string; customerName?: string;
}) {
  const widgetsRef = useRef<TossPaymentsWidgets | null>(null);
  const [ready, setReady] = useState(false);

  useEffect(() => {
    (async () => {
      const toss = await loadTossPayments(process.env.NEXT_PUBLIC_TOSS_CLIENT_KEY!);
      const widgets = toss.widgets({ customerKey: props.userId }); // 로그인 사용자 id
      await widgets.setAmount({ currency: "KRW", value: props.amount });
      await Promise.all([
        widgets.renderPaymentMethods({ selector: "#payment-method", variantKey: "DEFAULT" }),
        widgets.renderAgreement({ selector: "#agreement", variantKey: "AGREEMENT" }),
      ]);
      widgetsRef.current = widgets;
      setReady(true);
    })();
  }, [props.userId, props.amount]);

  const pay = async () => {
    await widgetsRef.current?.requestPayment({
      orderId: props.orderId,
      orderName: props.orderName,
      successUrl: `${window.location.origin}/checkout/success`,
      failUrl: `${window.location.origin}/checkout/fail`,
      customerEmail: props.customerEmail,
      customerName: props.customerName,
    });
  };

  return (
    <div>
      <div id="payment-method" />
      <div id="agreement" />
      <button disabled={!ready} onClick={pay}>
        {props.amount.toLocaleString("ko-KR")}원 결제하기
      </button>
    </div>
  );
}
```

## 5. 결제 승인 API — `app/api/payments/confirm/route.ts`

```ts
import { NextResponse } from "next/server";
import { createClient } from "@supabase/supabase-js";

export async function POST(req: Request) {
  const { paymentKey, orderId, amount } = await req.json();
  const admin = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  const { data: order } = await admin
    .from("orders").select("id, amount, status").eq("order_id", orderId).single();
  if (!order) return NextResponse.json({ message: "주문을 찾을 수 없어요" }, { status: 404 });
  if (order.status === "paid") return NextResponse.json({ ok: true });
  if (order.amount !== Number(amount)) {
    return NextResponse.json({ message: "결제 금액이 주문과 달라요" }, { status: 400 });
  }

  const auth = Buffer.from(`${process.env.TOSS_SECRET_KEY}:`).toString("base64");
  const res = await fetch("https://api.tosspayments.com/v1/payments/confirm", {
    method: "POST",
    headers: { Authorization: `Basic ${auth}`, "Content-Type": "application/json" },
    body: JSON.stringify({ paymentKey, orderId, amount: Number(amount) }),
  });
  const body = await res.json();
  if (!res.ok) return NextResponse.json(body, { status: res.status });

  await admin.from("orders")
    .update({ status: "paid", payment_key: paymentKey, paid_at: new Date().toISOString() })
    .eq("id", order.id);
  return NextResponse.json({ ok: true });
}
```

- 성공 페이지(`/checkout/success`)는 쿼리값을 읽어 이 API를 **한 번만** 호출하고, 결과에 따라 "결제 완료" 또는 오류를 보여준다.
- 실패 페이지(`/checkout/fail`)는 `code`, `message` 쿼리를 쉬운 말로 보여주고 "다시 시도" 버튼을 둔다.

## 6. 테스트
- 결제위젯에서 카드 결제 → 테스트 환경이라 실제 출금 없음
- 확인할 것: orders에 `paid`로 바뀌는지, `/my`에 4주 영상이 보이는지, 로그아웃 상태나 다른 계정으로는 안 보이는지

## 7. 실제 결제로 바꿀 때
1. 토스페이먼츠 상점 심사 통과
2. 개발자센터에서 **라이브 결제위젯 키**(`live_gck_`, `live_gsk_`) 확인
3. Vercel 환경변수 두 개를 라이브 키로 교체 → 재배포
4. 원장님 카드로 최소 금액 상품을 한 번 결제하고 취소해서 확인
