# Deep Dive: 데이터로그(Datalog) - 선언형 질의의 뿌리

2장에서 가장 난해하면서도 DDIA의 철학과 깊이 연결된 주제라고 느꼈습니다.

---

## 1. Prolog에서 Datalog로: 왜 제약을 뒀는가?

### Prolog란?

1972년 Alain Colmerauer가 개발한 **논리 프로그래밍 언어**입니다.

```prolog
% Prolog 예시: 가족 관계
parent(tom, bob).
% tom은 bob의 부모다.

parent(bob, ann).
% bob은 ann의 부모다.

grandparent(X, Z) :- parent(X, Y), parent(Y, Z).
% X가 Y의 부모고, Y가 Z의 부모면, X는 Z의 조부모이다. (:- 는 IF문)

% 질의: tom의 손자/손녀는?
?- grandparent(tom, Who).
% 결과: Who = ann
```

**Prolog의 특징:**
- 깊이 우선 탐색 방식
- 튜링 완전 (무한 루프 가능)
- 부정(negation)과 cut(!) 연산자 지원
- 함수 기호 허용 (복잡한 항 구성 가능)
- 실행 순서가 결과에 영향을 줌

### Prolog의 문제점

```prolog
% 무한 루프 발생 가능
ancestor(X, Y) :- ancestor(X, Z), parent(Z, Y).
ancestor(X, Y) :- parent(X, Y).

% 규칙 순서를 바꾸면 결과가 달라질 수 있음
```

| 문제 | 설명 |
|------|------|
| 종료 보장 없음 | 무한 루프에 빠질 수 있음 |
| 순서 의존성 | 규칙/절의 순서가 결과에 영향 |
| 병렬화 어려움 | 실행 순서가 중요해서 분산 처리 힘듦 |
| 최적화 한계 | 실행 계획을 시스템이 자유롭게 바꿀 수 없음 |

### Datalog의 탄생: 제약을 통한 자유

1970년대 후반~1980년대, 데이터베이스 연구자들이 Prolog에서 **"데이터베이스 질의에 필요한 것만"** 추출하여 만든 것이 Datalog입니다.

**Datalog의 핵심 제약:**

| 제약 | Prolog | Datalog | 이유 |
|------|--------|---------|------|
| 함수 기호 | 허용 | **금지** | 무한 항 생성 방지 → 종료 보장 |
| 부정 | 자유롭게 사용 | **계층화(Stratified)** | 안전한 의미론 보장 |
| 규칙 순서 | 결과에 영향 | **영향 없음** | 선언적 의미론 유지 |

```datalog
% Datalog: 순서가 바뀌어도 같은 결과
ancestor(X, Y) :- parent(X, Y).
ancestor(X, Y) :- ancestor(X, Z), parent(Z, Y).

% 위 두 줄의 순서를 바꿔도 동일한 결과!
```

### 제약이 가져다 준 것들

1. **종료 보장**: 모든 질의가 반드시 끝남
2. **순서 독립성**: 규칙을 어떤 순서로 써도 같은 결과
3. **병렬화 가능**: 실행 순서가 자유로워 분산 처리 용이
4. **최적화 자유**: 시스템이 최적의 실행 계획을 선택 가능

> **핵심 인사이트**: "제약을 걸어서 자유를 얻는다"  
> SQL이 명령형 대신 선언형을 선택한 것과 같은 철학입니다.

---

## 2. Datalog 예제로 이해하기

### 예제 1: DDIA 책의 그래프 질의

DDIA 2장에 나온 "미국 내에서 태어난 사람 찾기" 예제입니다.

**데이터 (사실, Facts):**

```datalog
% 장소 정보
name(1, "Idaho").
name(2, "United States").
name(3, "Lucy").

type(1, "state").
type(2, "country").
type(3, "person").

% 관계
within(1, 2).        % Idaho는 United States 안에 있다
born_in(3, 1).       % Lucy는 Idaho에서 태어났다
```

**규칙 (Rules):**

```datalog
% 재귀적 위치 관계: X가 Y 안에 있다
within_recursive(X, Y) :- within(X, Y).
within_recursive(X, Y) :- within(X, Z), within_recursive(Z, Y).

% 미국에서 태어난 사람
born_in_usa(Person) :-
    born_in(Person, Location),
    within_recursive(Location, USA),
    name(USA, "United States").
```

**질의:**

```datalog
?- born_in_usa(Person), name(Person, Name).
% 결과: Person = 3, Name = "Lucy"
```

### 예제 2: 소셜 네트워크 - 친구의 친구

```datalog
% 친구 관계 (양방향)
friend(alice, bob).
friend(bob, charlie).
friend(charlie, dave).
friend(alice, eve).

% 대칭성: A와 B가 친구면 B와 A도 친구
friend_symmetric(X, Y) :- friend(X, Y).
friend_symmetric(X, Y) :- friend(Y, X).

% 친구의 친구 (자기 자신과 직접 친구 제외)
friend_of_friend(X, Z) :-
    friend_symmetric(X, Y),
    friend_symmetric(Y, Z),
    X \= Z,
    not friend_symmetric(X, Z).
```

**질의:**

```datalog
?- friend_of_friend(alice, Who).
% 결과: Who = charlie, Who = dave (bob을 통해)
```

### 예제 3: 권한 시스템 - 재귀적 그룹 멤버십

실제 시스템에서 자주 쓰이는 패턴입니다.

```datalog
% 직접 멤버십
direct_member(alice, engineering).
direct_member(bob, engineering).
direct_member(engineering, tech_org).
direct_member(tech_org, company).

% 재귀적 멤버십: 그룹의 그룹도 포함
member(X, G) :- direct_member(X, G).
member(X, G) :- direct_member(X, H), member(H, G).

% 권한 부여
has_permission(Group, read, repo_a) :- member(Group, tech_org).
has_permission(Group, write, repo_a) :- member(Group, engineering).

% 사용자의 권한 확인
can_access(User, Action, Resource) :-
    member(User, Group),
    has_permission(Group, Action, Resource).
```

**질의:**

```datalog
?- can_access(alice, write, repo_a).
% 결과: true (alice → engineering → has write permission)

?- can_access(alice, read, repo_a).
% 결과: true (alice → engineering → tech_org → has read permission)
```

### SQL과의 비교

같은 질의를 SQL로 작성하면:

```sql
-- 재귀적 위치 관계 (SQL WITH RECURSIVE)
WITH RECURSIVE within_recursive AS (
    SELECT head_id AS location, tail_id AS contained_in
    FROM edges WHERE label = 'within'
    
    UNION
    
    SELECT wr.location, e.tail_id
    FROM within_recursive wr
    JOIN edges e ON wr.contained_in = e.head_id
    WHERE e.label = 'within'
)
SELECT v.name
FROM vertices v
JOIN edges born ON born.tail_id = v.id AND born.label = 'born_in'
JOIN within_recursive wr ON born.head_id = wr.location
JOIN vertices usa ON wr.contained_in = usa.id AND usa.name = 'United States';
```

| 비교 항목 | Datalog | SQL WITH RECURSIVE |
|----------|---------|-------------------|
| 코드 길이 | ~10줄 | ~20줄 |
| 가독성 | 선언적, 의도 명확 | 절차적 느낌 |
| 재귀 표현 | 자연스러움 | UNION 필요 |
| 최적화 | 자동 | 제한적 |

---

## 3. 규칙 = 함수: 함수형 프로그래밍과의 연결

### Datalog 규칙을 함수로 보기

```datalog
ancestor(X, Y) :- parent(X, Y).
ancestor(X, Y) :- ancestor(X, Z), parent(Z, Y).
```

이것을 함수형 스타일로 표현하면:

```javascript
// JavaScript 함수형 스타일
const ancestor = (x, y, facts) => {
  // Base case: 직접 부모
  if (facts.parent.has([x, y])) return true;
  
  // Recursive case: 조상의 자식
  return facts.parent
    .filter(([a, b]) => b === y)
    .some(([z, _]) => ancestor(x, z, facts));
};
```

```haskell
-- Haskell 스타일
ancestor :: Person -> Person -> [Fact] -> Bool
ancestor x y facts
  | (x, y) `elem` parents facts = True
  | otherwise = any (\z -> ancestor x z facts) (parentsOf y facts)
```

### 공통점 분석

| 개념 | Datalog | 함수형 프로그래밍 |
|------|---------|------------------|
| 기본 단위 | 규칙 (Rule) | 함수 (Function) |
| 재귀 | 규칙이 자신을 참조 | 함수가 자신을 호출 |
| 부작용 | 없음 | 없음 (순수 함수) |
| 평가 순서 | 시스템이 결정 | Lazy evaluation |
| 합성 | 규칙 체이닝 | 함수 합성 |

### 고차 함수와의 대응

```datalog
% Datalog: 필터링
adult(X) :- person(X), age(X, A), A >= 18.

% Datalog: 매핑 (새로운 관계 생성)
full_name(X, Full) :- 
    first_name(X, First), 
    last_name(X, Last),
    concat(First, " ", Last, Full).

% Datalog: 집계 (확장 문법)
team_size(Team, Count) :- 
    Count = count : { member(_, Team) }.
```

```javascript
// 함수형 JavaScript 대응
const adults = people.filter(p => p.age >= 18);

const fullNames = people.map(p => ({
  ...p,
  fullName: `${p.firstName} ${p.lastName}`
}));

const teamSizes = teams.map(team => ({
  team,
  count: members.filter(m => m.team === team).length
}));
```

### 핵심 인사이트: 선언적 사고

```
명령형: "어떻게(How) 할 것인가"
├── for 루프로 순회
├── if 조건으로 필터
└── 결과 배열에 push

선언형: "무엇(What)을 원하는가"  
├── Datalog: 규칙으로 관계 정의
├── SQL: SELECT로 원하는 것 명시
└── 함수형: map/filter/reduce로 변환 명시
```

> **DDIA의 메시지**: "선언형이 좋은 이유는 최적화를 시스템에 맡길 수 있기 때문"

---

## 4. 분산 시스템에서의 선언형 접근

### 왜 분산 시스템에서 Datalog가 주목받는가?

Datalog의 특성이 분산 환경에 잘 맞습니다:

| 특성 | 분산 시스템에서의 이점 |
|------|----------------------|
| 순서 독립성 | 여러 노드에서 병렬 실행 가능 |
| 종료 보장 | 무한 루프로 클러스터가 죽지 않음 |
| 단조성(Monotonicity) | 새 데이터 추가해도 기존 결과 유효 |
| 선언적 명세 | 분산 로직을 간결하게 표현 |

### Bloom 언어: 분산 시스템을 위한 Datalog

UC Berkeley에서 개발한 **Bloom**은 Datalog를 분산 프로그래밍에 적용한 언어입니다.

```ruby
# Bloom 예시: 분산 키-값 저장소
module KVStore
  state do
    table :store, [:key] => [:value]
    channel :request, [:@addr, :key, :type]
    channel :response, [:@addr, :key, :value]
  end

  bloom do
    # GET 요청 처리
    response <~ (request * store).pairs(:key => :key) {|r, s|
      [r.addr, r.key, s.value] if r.type == :get
    }
    
    # PUT 요청 처리
    store <+ request {|r| [r.key, r.value] if r.type == :put }
  end
end
```

### CALM 정리: 일관성과 논리의 연결

Bloom 연구에서 나온 **CALM (Consistency As Logical Monotonicity)** 정리:

> "단조적(monotonic) 프로그램은 coordination 없이도 eventually consistent하다"

**단조성이란?**
- 새로운 사실이 추가되어도 기존 결론이 철회되지 않음
- 예: "A가 B의 조상이다"라는 결론은 새 데이터가 와도 유지됨

```datalog
% 단조적 (Monotonic) - 새 사실 추가해도 안전
reachable(X, Y) :- edge(X, Y).
reachable(X, Y) :- reachable(X, Z), edge(Z, Y).

% 비단조적 (Non-monotonic) - 새 사실이 결과를 바꿀 수 있음
not_reachable(X, Y) :- node(X), node(Y), NOT reachable(X, Y).
```

| 연산 타입 | 예시 | Coordination 필요? |
|----------|------|-------------------|
| 단조적 | 합집합, 존재 확인 | 불필요 |
| 비단조적 | 집계, 부정, 전체 확인 | 필요 |

### 현대 시스템에서의 적용

#### 1. Datomic (Clojure 생태계)

```clojure
;; Datomic Datalog 질의
(d/q '[:find ?name
       :where
       [?e :person/name ?name]
       [?e :person/age ?age]
       [(> ?age 21)]]
     db)
```

- 불변 데이터베이스
- 시간 여행 쿼리 (과거 상태 조회)
- Datalog 기반 질의

#### 2. Differential Dataflow (Materialize, Timely Dataflow)

- 증분 계산: 데이터 변경 시 전체 재계산 없이 결과 갱신
- 실시간 뷰 유지에 활용
- Datalog 의미론 기반

#### 3. LogicBlox / RelationalAI

- 기업용 분석 플랫폼
- 최적화 문제, 규칙 기반 추론
- 대규모 Datalog 실행 엔진

### 분산 시스템 설계 패턴으로서의 Datalog

```
전통적 분산 시스템 설계:
├── 메시지 전송
├── 타임아웃 처리  
├── 재시도 로직
├── 상태 동기화
└── 에러 핸들링

Datalog 기반 설계:
├── 원하는 상태를 규칙으로 선언
├── 시스템이 알아서 수렴
└── 비단조적 부분만 coordination 추가
```
