# HBase

## HBase API

HBase는 다양한 클라이언트 API를 제공하며, Python에서는 **happybase** 라이브러리를 통해 Thrift 기반으로 접근합니다.

### 연결 설정

```python
import happybase

connection = happybase.Connection('localhost', port=9090)
table = connection.table('my_table')
```

- `Connection`은 Thrift 서버를 통해 HBase에 연결합니다
- `table()`로 테이블 객체를 가져와 모든 CRUD를 수행합니다

---

## CRUD 연산

### 1. Put (쓰기)

단일 row에 데이터를 삽입하거나 업데이트합니다.

```python
table.put(b'row1', {
    b'cf:name': b'Alice',
    b'cf:age': b'25',
})
```

**배치 쓰기**도 가능합니다.

```python
batch = table.batch()
batch.put(b'row1', {b'cf:name': b'Alice'})
batch.put(b'row2', {b'cf:name': b'Bob'})
batch.put(b'row3', {b'cf:name': b'Charlie'})
batch.send()
```

| 옵션 | 설명 |
|---|---|
| `timestamp=` | 특정 타임스탬프 지정 |
| `wal=False` | WAL 쓰기 비활성화 (속도 향상, 내구성 감소) |
| `batch(transaction=True)` | 배치를 원자적으로 실행 |

### 2. Get (단건 읽기)

row key로 단일 row를 조회합니다.

```python
# 전체 컬럼 조회
row = table.row(b'row1')
print(row[b'cf:name'])  # b'Alice'

# 특정 컬럼만 조회
row = table.row(b'row1', columns=[b'cf:name', b'cf:age'])

# 여러 row 한번에 조회
rows = table.rows([b'row1', b'row2', b'row3'])
for key, data in rows:
    print(key, data)
```

| 옵션 | 설명 |
|---|---|
| `columns=[...]` | 특정 컬럼만 조회 |
| `timestamp=` | 특정 시점의 데이터 조회 |
| `include_timestamp=True` | 타임스탬프 함께 반환 |

### 3. Scan (범위 읽기)

row key 범위를 지정하여 여러 row를 순회합니다.

```python
# 범위 스캔
for key, data in table.scan(row_start=b'row1', row_stop=b'row9'):
    print(key, data)

# prefix 스캔
for key, data in table.scan(row_prefix=b'user_'):
    print(key, data)

# 필터 + 제한
for key, data in table.scan(
    row_start=b'row1',
    columns=[b'cf:name'],
    filter=b"SingleColumnValueFilter('cf','age',>=,'binary:20')",
    limit=100,
):
    print(key, data)
```

| 옵션 | 설명 |
|---|---|
| `row_start` / `row_stop` | 스캔 범위 (start 포함, stop 미포함) |
| `row_prefix` | row key prefix 매칭 |
| `columns=[...]` | 특정 컬럼만 조회 |
| `filter=` | 서버 측 필터 문자열 |
| `limit=n` | 최대 반환 row 수 |
| `reverse=True` | 역방향 스캔 |
| `batch_size=n` | 한 번의 RPC로 가져올 row 수 |

### 4. Delete (삭제)

```python
# row 전체 삭제
table.delete(b'row1')

# 특정 컬럼만 삭제
table.delete(b'row1', columns=[b'cf:name'])

# 배치 삭제
batch = table.batch()
batch.delete(b'row1')
batch.delete(b'row2')
batch.send()
```

> HBase의 삭제는 즉시 물리 삭제가 아니라 **tombstone marker**를 남기고, 이후 **Major Compaction** 시 실제 제거됩니다.

---

## 고급 API

### 5. Counter (원자적 증가)

카운터 용도로 사용합니다. 별도의 락 없이 원자적으로 동작합니다.

```python
# 카운터 증가
table.counter_inc(b'row1', b'cf:view_count', value=1)

# 카운터 감소
table.counter_dec(b'row1', b'cf:view_count', value=1)

# 현재 카운터 값 조회
count = table.counter_get(b'row1', b'cf:view_count')
print(count)  # 42

# 카운터 값 직접 설정
table.counter_set(b'row1', b'cf:view_count', 0)
```

### 6. Batch (혼합 배치)

Put과 Delete를 하나의 배치로 묶어 실행합니다.

```python
batch = table.batch()
batch.put(b'row1', {b'cf:status': b'inactive'})
batch.put(b'row2', {b'cf:status': b'active'})
batch.delete(b'row3')
batch.send()
```

context manager를 사용하면 자동으로 `send()`가 호출됩니다.

```python
with table.batch(batch_size=1000) as batch:
    for i in range(10000):
        batch.put(f'row{i}'.encode(), {
            b'cf:data': f'value{i}'.encode(),
        })
    # batch_size(1000)마다 자동 flush, 마지막에 자동 send
```

---

## Filter API

Scan에 서버 측 필터를 문자열로 전달하여 네트워크 전송량을 줄일 수 있습니다.

| 필터 | 문법 예시 |
|---|---|
| `SingleColumnValueFilter` | `"SingleColumnValueFilter('cf','age',>=,'binary:20')"` |
| `PrefixFilter` | `"PrefixFilter('user_')"` |
| `RowFilter` | `"RowFilter(=,'regexstring:user_.*')"` |
| `PageFilter` | `"PageFilter(100)"` |
| `ColumnPrefixFilter` | `"ColumnPrefixFilter('addr_')"` |
| 필터 조합 (AND) | `"SingleColumnValueFilter(...) AND PrefixFilter(...)"` |
| 필터 조합 (OR) | `"SingleColumnValueFilter(...) OR PrefixFilter(...)"` |

```python
# 나이가 20 이상인 row만 스캔
for key, data in table.scan(
    filter=b"SingleColumnValueFilter('cf','age',>=,'binary:20')",
):
    print(key, data)

# 여러 필터 조합
filter_str = (
    b"SingleColumnValueFilter('cf','status',=,'binary:active') "
    b"AND ColumnPrefixFilter('addr_')"
)
for key, data in table.scan(filter=filter_str):
    print(key, data)
```

---

## 테이블 관리 API

```python
connection = happybase.Connection('localhost')

# 테이블 목록 조회
print(connection.tables())  # [b'my_table', b'users']

# 테이블 생성
connection.create_table('users', {
    'cf': dict(max_versions=3, bloom_filter_type='ROW'),
})

# 테이블 비활성화 / 삭제
connection.disable_table('users')
connection.delete_table('users')

# 테이블 활성화 여부 확인
connection.is_table_enabled('users')
```

---

## API 호출 흐름 요약

```
Client API (happybase)
    ↓
Thrift Server (HBase Thrift Gateway)
    ↓
HBase Master / RegionServer (해당 Region에서 처리)
    ↓
Region → MemStore (쓰기) / MemStore + StoreFile (읽기)
```

핵심 포인트:
- **쓰기**: WAL 기록 → MemStore 반영 → flush 시 HFile로 디스크 저장
- **읽기**: MemStore + StoreFile을 병합하여 최신 데이터 반환
- **모든 연산은 row 단위로 원자적**입니다 (단일 row 내 여러 컬럼 수정은 atomic)
- Python(happybase)은 Thrift를 경유하므로 Java 네이티브 API 대비 약간의 오버헤드가 있습니다
