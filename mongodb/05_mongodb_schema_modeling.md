# MongoDB Schema Modeling

## 개요

MongoDB는 스키마리스(Schema-less) 데이터베이스지만, 실제 서비스에서는 **데이터 접근 패턴(Access Pattern)** 을 기반으로 스키마를 설계해야 한다.
핵심 원칙: **"How you use your data should determine how you store it"**

---

## 핵심 설계 원칙

| 원칙 | 설명 |
|------|------|
| **Embed vs Reference** | 함께 조회되는 데이터는 내장(Embed), 독립적으로 사용되는 데이터는 참조(Reference) |
| **Write vs Read 비율** | 읽기가 많으면 비정규화(중복 저장), 쓰기가 많으면 정규화(참조) |
| **Document 크기** | 16MB 제한, 자주 커지는 배열은 별도 컬렉션으로 분리 |
| **Cardinality** | 1:1, 1:N, N:M 관계에 따라 전략이 다름 |

---

## Embedding vs Referencing

### Embedding (내장)
```json
// 주문 안에 상품 정보를 내장
{
  "_id": "order_001",
  "userId": "user_123",
  "items": [
    { "productId": "p1", "name": "노트북", "price": 1200000, "qty": 1 },
    { "productId": "p2", "name": "마우스", "price": 30000, "qty": 2 }
  ],
  "total": 1260000,
  "createdAt": "2024-01-15"
}
```
- 장점: 단일 쿼리로 모든 데이터 조회
- 단점: 상품 정보 변경 시 모든 주문 도큐먼트 업데이트 필요

### Referencing (참조)
```json
// 주문은 상품 ID만 참조
{
  "_id": "order_001",
  "userId": "user_123",
  "items": [
    { "productId": "p1", "qty": 1 },
    { "productId": "p2", "qty": 2 }
  ]
}

// 상품 컬렉션 별도 관리
{
  "_id": "p1",
  "name": "노트북",
  "price": 1200000,
  "stock": 50
}
```
- 장점: 데이터 일관성 유지, 중복 없음
- 단점: $lookup(JOIN) 필요, 쿼리 복잡도 증가

---

## 서비스별 스키마 모델링 사례

---

### 1. E-Commerce (쇼핑몰)

#### 요구사항
- 상품 목록 조회 (빠른 읽기)
- 주문 내역 조회
- 리뷰 작성 및 조회
- 재고 관리

#### 상품(Product) 스키마
```json
{
  "_id": "product_001",
  "name": "맥북 프로 14인치",
  "category": ["전자기기", "노트북"],
  "price": 2990000,
  "stock": 100,
  "attributes": {
    "brand": "Apple",
    "color": "스페이스 그레이",
    "weight": "1.6kg",
    "specs": {
      "cpu": "M3 Pro",
      "memory": "18GB",
      "storage": "512GB SSD"
    }
  },
  "images": [
    "https://cdn.example.com/products/p001_1.jpg",
    "https://cdn.example.com/products/p001_2.jpg"
  ],
  "rating": { "avg": 4.7, "count": 328 },  // 비정규화 - 매번 집계 방지
  "tags": ["애플", "노트북", "M3"],
  "createdAt": "2024-01-01",
  "updatedAt": "2024-03-01"
}
```

#### 주문(Order) 스키마 - 주문 시점 가격 스냅샷
```json
{
  "_id": "order_20240315_001",
  "userId": "user_123",
  "status": "delivered",  // pending → paid → shipped → delivered
  "items": [
    {
      "productId": "product_001",
      "name": "맥북 프로 14인치",      // 스냅샷: 주문 당시 상품명
      "price": 2990000,                // 스냅샷: 주문 당시 가격
      "qty": 1,
      "subtotal": 2990000
    }
  ],
  "shippingAddress": {
    "name": "홍길동",
    "phone": "010-1234-5678",
    "address": "서울시 강남구 테헤란로 123",
    "zipCode": "06234"
  },
  "payment": {
    "method": "card",
    "amount": 2990000,
    "paidAt": "2024-03-15T10:30:00Z"
  },
  "createdAt": "2024-03-15T10:29:00Z"
}
```

> 핵심 포인트: 상품 가격이 나중에 바뀌어도 주문 내역은 구매 당시 가격을 보존해야 하므로 **스냅샷 패턴** 사용

#### 리뷰(Review) - 별도 컬렉션 (unbounded array 방지)
```json
{
  "_id": "review_001",
  "productId": "product_001",
  "userId": "user_123",
  "userName": "홍*동",          // 비정규화 - 표시용
  "rating": 5,
  "title": "정말 빠르고 가볍네요",
  "content": "업무용으로 최고입니다. 배터리도 오래가고...",
  "images": ["https://cdn.example.com/reviews/r001_1.jpg"],
  "helpful": 42,
  "createdAt": "2024-03-20T14:00:00Z"
}
```

> 리뷰가 무한정 늘어날 수 있으므로 Product 도큐먼트에 내장하지 않고 별도 컬렉션으로 관리. Product에는 `rating.avg`, `rating.count`만 비정규화하여 저장.

---

### 2. SNS / 소셜 미디어

#### 요구사항
- 피드 게시글 작성 및 조회
- 팔로우/팔로워 관계
- 좋아요, 댓글
- 해시태그 검색

#### 사용자(User) 스키마
```json
{
  "_id": "user_123",
  "username": "gildong_hong",
  "displayName": "홍길동",
  "email": "gildong@example.com",
  "bio": "개발자 | 커피 애호가",
  "profileImage": "https://cdn.example.com/avatars/user_123.jpg",
  "stats": {
    "posts": 142,
    "followers": 3820,
    "following": 251
  },
  "createdAt": "2023-06-01"
}
```

#### 팔로우 관계 - 별도 컬렉션 (N:M 관계)
```json
// follows 컬렉션
{
  "_id": "follow_001",
  "followerId": "user_123",   // 팔로우하는 사람
  "followeeId": "user_456",   // 팔로우 당하는 사람
  "createdAt": "2024-01-10"
}

// 인덱스: { followerId: 1 }, { followeeId: 1 }
```

#### 게시글(Post) 스키마
```json
{
  "_id": "post_001",
  "authorId": "user_123",
  "author": {                           // 비정규화 - 피드 조회 최적화
    "username": "gildong_hong",
    "displayName": "홍길동",
    "profileImage": "https://cdn.example.com/avatars/user_123.jpg"
  },
  "content": "오늘 MongoDB 공부 완료! #MongoDB #NoSQL #개발",
  "images": ["https://cdn.example.com/posts/post_001_1.jpg"],
  "hashtags": ["MongoDB", "NoSQL", "개발"],
  "likes": {
    "count": 284,
    "recentUsers": ["user_456", "user_789", "user_101"]  // 최근 좋아요 3명만 저장
  },
  "comments": {
    "count": 17                          // 댓글 수만 저장, 실제 댓글은 별도 컬렉션
  },
  "visibility": "public",
  "createdAt": "2024-03-15T09:00:00Z"
}
```

#### 댓글(Comment) - 별도 컬렉션
```json
{
  "_id": "comment_001",
  "postId": "post_001",
  "authorId": "user_456",
  "author": {
    "username": "jane_doe",
    "profileImage": "https://cdn.example.com/avatars/user_456.jpg"
  },
  "content": "저도 오늘 공부했어요!",
  "likes": 5,
  "parentId": null,                 // null이면 최상위 댓글, 있으면 대댓글
  "createdAt": "2024-03-15T09:30:00Z"
}
```

#### 좋아요(Like) - 별도 컬렉션 (중복 방지 및 취소 지원)
```json
{
  "_id": "like_001",
  "targetType": "post",           // "post" | "comment"
  "targetId": "post_001",
  "userId": "user_789",
  "createdAt": "2024-03-15T10:00:00Z"
}

// 유니크 인덱스: { targetId: 1, userId: 1 }
```

---

### 3. 채팅 / 메시징 서비스

#### 요구사항
- 1:1 채팅, 그룹 채팅
- 메시지 전송 및 조회 (무한 스크롤)
- 읽음 처리
- 마지막 메시지 표시

#### 채팅방(ChatRoom) 스키마
```json
{
  "_id": "room_001",
  "type": "group",                      // "direct" | "group"
  "name": "MongoDB 스터디",
  "members": [
    {
      "userId": "user_123",
      "role": "admin",
      "joinedAt": "2024-01-01",
      "lastReadAt": "2024-03-15T10:00:00Z"  // 읽음 처리용
    },
    {
      "userId": "user_456",
      "role": "member",
      "joinedAt": "2024-01-02",
      "lastReadAt": "2024-03-15T09:45:00Z"
    }
  ],
  "lastMessage": {                      // 비정규화 - 채팅방 목록 표시 최적화
    "content": "내일 스터디 몇 시에요?",
    "senderId": "user_456",
    "sentAt": "2024-03-15T09:45:00Z"
  },
  "createdAt": "2024-01-01"
}
```

#### 메시지(Message) - 버킷 패턴 적용
```json
// 하루치 메시지를 하나의 도큐먼트에 버킷
{
  "_id": "bucket_room001_20240315",
  "roomId": "room_001",
  "date": "2024-03-15",
  "count": 47,
  "messages": [
    {
      "msgId": "msg_001",
      "senderId": "user_123",
      "type": "text",                   // "text" | "image" | "file"
      "content": "안녕하세요!",
      "sentAt": "2024-03-15T08:00:00Z",
      "status": "read"
    },
    {
      "msgId": "msg_002",
      "senderId": "user_456",
      "type": "image",
      "content": "https://cdn.example.com/chat/img_001.jpg",
      "sentAt": "2024-03-15T08:01:00Z",
      "status": "delivered"
    }
    // ... 더 많은 메시지
  ]
}
```

> **버킷 패턴(Bucket Pattern)**: 시계열 데이터나 메시지처럼 대량으로 쌓이는 데이터를 날짜/시간 단위로 묶어 저장. 도큐먼트 수를 줄이고 조회 성능을 향상.

---

### 4. 콘텐츠 관리 시스템 (CMS / 블로그)

#### 요구사항
- 게시글 작성 (마크다운, 버전 관리)
- 카테고리 / 태그 분류
- 댓글
- 조회수 통계

#### 게시글(Article) 스키마
```json
{
  "_id": "article_001",
  "slug": "mongodb-schema-design-guide",     // URL 친화적 ID
  "title": "MongoDB 스키마 설계 가이드",
  "content": "# MongoDB 스키마 설계\n\n...",  // Markdown
  "excerpt": "MongoDB 스키마 설계의 핵심 원칙과 실전 예시를 알아봅니다.",
  "author": {
    "userId": "user_123",
    "name": "홍길동",
    "avatar": "https://cdn.example.com/avatars/user_123.jpg"
  },
  "category": "Database",
  "tags": ["MongoDB", "NoSQL", "Schema Design"],
  "status": "published",                     // "draft" | "published" | "archived"
  "thumbnail": "https://cdn.example.com/articles/thumb_001.jpg",
  "stats": {
    "views": 15420,
    "likes": 342,
    "comments": 28,
    "shares": 95
  },
  "seo": {
    "metaTitle": "MongoDB 스키마 설계 가이드 | DevBlog",
    "metaDescription": "실전 예시로 배우는 MongoDB 스키마 설계"
  },
  "publishedAt": "2024-03-01T10:00:00Z",
  "updatedAt": "2024-03-10T15:00:00Z"
}
```

#### 버전 관리 - 별도 컬렉션
```json
{
  "_id": "version_001",
  "articleId": "article_001",
  "version": 3,
  "content": "# MongoDB 스키마 설계\n\n이전 버전 내용...",
  "changedBy": "user_123",
  "createdAt": "2024-03-05T12:00:00Z"
}
```

---

### 5. IoT / 시계열 데이터

#### 요구사항
- 센서 데이터 초당 수집
- 시간대별 집계 (시간/일/월 평균)
- 최근 N시간 데이터 조회

#### 버킷 패턴 - 1시간 단위 집계
```json
{
  "_id": "sensor_A01_2024031510",
  "sensorId": "sensor_A01",
  "location": "서울 강남 측정소",
  "hour": "2024-03-15T10:00:00Z",          // 버킷 시작 시간
  "count": 3600,                            // 측정 횟수
  "measurements": {
    "temperature": {
      "min": 18.2,
      "max": 23.5,
      "avg": 20.8,
      "sum": 74880
    },
    "humidity": {
      "min": 45.0,
      "max": 67.3,
      "avg": 55.2,
      "sum": 198720
    }
  },
  "readings": [                             // 원본 데이터 (선택적 저장)
    { "ts": "2024-03-15T10:00:01Z", "temp": 20.1, "humidity": 55.3 },
    { "ts": "2024-03-15T10:00:02Z", "temp": 20.2, "humidity": 55.1 }
    // ...
  ]
}
```

---

## 주요 패턴 정리

### Subset Pattern (서브셋 패턴)
자주 접근하는 데이터만 메인 도큐먼트에 저장
```json
// Product에 최근 리뷰 5개만 내장, 나머지는 reviews 컬렉션 조회
{
  "_id": "product_001",
  "name": "맥북 프로",
  "recentReviews": [              // 최신 5개만 유지
    { "rating": 5, "content": "최고예요!", "userId": "u1" }
  ],
  "rating": { "avg": 4.7, "count": 328 }
}
```

### Computed Pattern (사전 계산 패턴)
읽기 시 집계 연산을 피하기 위해 미리 계산된 값을 저장
```json
// 게시글 조회수: 매번 COUNT 쿼리 대신 stats.views 필드 직접 증가
db.articles.updateOne(
  { _id: "article_001" },
  { $inc: { "stats.views": 1 } }
)
```

### Outlier Pattern (아웃라이어 패턴)
대부분의 경우는 내장, 예외적으로 큰 경우만 별도 처리
```json
{
  "_id": "post_001",
  "likes": {
    "count": 10000000,
    "hasOverflow": true,          // 좋아요 수가 너무 많아 별도 컬렉션 사용
    "recentUsers": ["u1", "u2"]  // 최근 몇 명만 내장
  }
}
```

---

## 안티 패턴 (피해야 할 것들)

| 안티 패턴 | 문제점 | 해결책 |
|-----------|--------|--------|
| **Massive Array** | 배열이 무한정 커짐 (16MB 제한) | 별도 컬렉션으로 분리 |
| **Bloated Document** | 불필요한 필드가 너무 많아 항상 큰 도큐먼트 읽기 | 필드 선택적 프로젝션 or 컬렉션 분리 |
| **Unnecessary Indexes** | 인덱스가 너무 많으면 쓰기 성능 저하 | 실제 쿼리 패턴 기반으로만 인덱스 생성 |
| **Relational Thinking** | MongoDB를 RDB처럼 모든 것을 정규화 | 접근 패턴 기반으로 비정규화 활용 |

---

## 스키마 설계 체크리스트

- [ ] 주요 쿼리 패턴(읽기/쓰기 비율)을 정의했는가?
- [ ] 배열 필드가 무한정 커질 가능성이 없는가?
- [ ] 도큐먼트 크기가 16MB를 초과할 위험이 없는가?
- [ ] 자주 함께 조회되는 데이터는 내장(Embed)했는가?
- [ ] 독립적으로 자주 수정되는 데이터는 참조(Reference)했는가?
- [ ] 읽기 성능을 위해 필요한 필드는 비정규화(중복 저장)했는가?
- [ ] 적절한 인덱스가 설계되었는가?