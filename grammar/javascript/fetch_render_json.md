## Fetch -> JSON 데이터 가공 -> 최종 출력


### 1. 서버 응답 (JSON)

[
  {
    "id": 1,
    "productName": "나이키 에어포스",
    "productMgtId": "P-1001",
    "productThumbnailUrl": "/images/nike.png",
    "price": 120000,
    "count": 2,
    "size": "270"
  },
  {
    "id": 2,
    "productName": "아디다스 슈퍼스타",
    "productMgtId": "P-1002",
    "productThumbnailUrl": null,
    "price": 99000,
    "count": 1,
    "size": "260"
  }
]

### 2. fetchCartItems() 

function fechCartItems() {
    fetch("/api/v1/cart/my", { headers: { Authorization: "Bearer 토큰값" }})
        .then(res => res.json())
        .then(renderCart) // 데이터 넘김
        .then(() => calculatePrice())
        .catch(err => {...});
}

-> 서버에서 장바구니 JSON 데이터를 가져오고 renderCart()로 넘김

### 3. renderCart(cartItems)

cartItems.forEach(item => {
    const div = document.createElement('div');
    div.className = 'cart-item';
    div.innerHTML = '
    <input type="checkbox" class="item-select" data-cart-id="${item.id}" />
    <img src="${item.productThumbnailUrl || '/images/default-product.png'}" alt="${item.productName}" class="product-img" />
    <div class="product-info">
      <strong>${item.productName}</strong><br />
      <small>사이즈: ${item.size || '-'}</small><br />
      <small>상품 코드: ${item.productMgtId}</small>
    </div>
    <div class="price-info">
      <span>${Number(item.price).toLocaleString()}원</span>
      <input type="number" min="1" value="${item.count}" onchange="updateCount(${item.id}, this.value)" />
      <button onclick="deleteCartItem(${item.id})">삭제</button>
    </div>
  `;
  container.appendChild(div);
});

-> JSON 객체 -> HTML DOM 요소로 변환해서 <div class="cart-item">...</div> 형태로 페이지에 붙임

### 4. 실제 화면 결과

[ ] (이미지: 나이키 에어포스)
    나이키 에어포스
    사이즈: 270
    상품 코드: P-1001
    가격: 120,000원   [수량: 2]  [삭제버튼]

[ ] (이미지: default-product.png)
    아디다스 슈퍼스타
    사이즈: 260
    상품 코드: P-1002
    가격: 99,000원   [수량: 1]  [삭제버튼]

### 5. calculatePrice()

function calculatePrice() {
    const items = document.querySelectorAll('.cart-item');
    let total = 0;

    items.forEach(item => {
        const count = item.querySelector("input[type=number]").value;
        const priceText = item.querySelector(".price-info span").innerText; // "120,000원"
        const price = parseInt(priceText.replace(/[^0-9]/g, "")); // 숫자만 추출

    total += price * count;
  });

  document.getElementById("total-price").innerText = total.toLocaleString() + "원";
}

### 6. 최종 결과 흐름

[ 서버 JSON ]
[
  { id:1, name:"나이키", price:120000, count:2, ... },
  { id:2, name:"아디다스", price:99000, count:1, ... }
]
      │
      ▼
fetchCartItems()
      │
      ▼
renderCart(cartItems)
      │  (DOM 변환)
      ▼
[HTML]
<div class="cart-item"> ... </div>
<div class="cart-item"> ... </div>
      │
      ▼
calculatePrice()
      │
      ▼
총합 = 120000×2 + 99000×1 = 339,000원
      │
      ▼
#total-price = "339,000원"

-> 이렇게 해서 서버 JSON → DOM 변환 → 총합 계산 → 화면 표시



