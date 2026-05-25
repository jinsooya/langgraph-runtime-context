# LangGraph Runtime Context

## 1. Runtime Context란?

LangGraph **Runtime Context**는 **그래프 상태(State)와 별도로**, 실행할 때마다 **외부 리소스·설정을 주입**하는 메커니즘이다.
- 대화 내용(`messages` 등)은 State에 담고, 
- DB 연결, LLM 인스턴스, API 클라이언트, 사용자 프로필, 실행 환경 설정처럼 **노드 및 도구가 공유하는 의존성**은 Runtime Context에 담는다.

| | 내용 |
|---|---|
| **Runtime Context 사용하는 이유** | State와 분리한 DI, 시그니처 단순화, 실행마다 다른 설정, 타입 및 구조 명시 |
| **Runtime Context 단점** | 그래프 밖 테스트 어려움, 암묵적 의존성, Memory와 혼동, 스키마(이름과 타입 정의)를 썼다고 해서, context에 들어가는 값이 전부 검증된다는 뜻은 아니다 |


프로젝트마다 context 스키마 이름과 필드는 다르지만, LangGraph에서는 보통 **타입(또는 스키마 클래스)** 를 정의하고 `context_schema`와 `context=`로 연결한다.

```python
# 예시 (필드 이름은 프로젝트마다 다름)
runtime = get_runtime(MyContextSchema)
resource = runtime.context.some_dependency

graph.invoke(input=..., context=MyContextSchema(...))
```

## 2. State / Memory / Runtime Context 구분

| 구분 | 역할 | 예시 |
|------|------|------|
| **State** | 실행 흐름 안에서 변하는 그래프 데이터 | `messages`, 중간 결과 |
| **Runtime Context** | 실행 시 개발자가 주입하는 의존성 또는 설정 | DB, LLM, API 키, 사용자 역할 |
| **Memory (checkpointer)** | `thread_id` 등으로 State를 지속 | 대화 기록 복원, 멀티턴 |

> Memory와 Runtime Context는 둘 다 '기억'처럼 들리지만, **역할이 다르다.**

- - -

## 3. 사용하는 이유 (장점)

### (1) State와 역할을 분리한다

- **State**: 턴마다 쌓이는 대화 및 중간 산출물을 담는다.
- **Runtime Context**: 실행 환경과 공유 리소스를 담는다.

같은 리소스로 여러 요청을 처리할 때, State마다 객체를 복제해 넣을 필요가 없다.
- 예) 같은 DB로 여러 질문을 처리할 때, 메시지 state마다 DB 객체를 넣을 필요가 없다.

### (2) 의존성 주입(dependency injection, DI)을 적용한다

노드와 도구가 전역 변수에 묶이지 않고, 실행 시점에 주입한 리소스만 사용한다.

- 테스트와 재사용이 쉬워진다.
- 환경(개발/스테이징/프로덕션)을 바꾸기 쉽다.
    - 예) 다른 DB 또는 다른 모델로 환경을 바꾸기 쉽다.
- 호출부 `context=...`에서 의존성을 명시한다.

### (3) 도구나 노드 시그니처를 단순화한다

- LLM이 보는 도구 전달 인자에는 **업무 데이터만** 둔다.  
    - 예) LLM이 호출하는 도구 전달인자에는 `query`, `table_names` 같은 **업무 데이터만** 둔다. 
- 인프라와 리소스는 context로 전달한다.
    - 예) `db`와 `model`은 context로 전달한다.
- 도구 정의가 깔끔해진다.
- LLM이 내부 객체(예: DB 핸들)를 전달인자로 넘기는 실수가 줄어든다.

### (4) 실행마다 다른 설정을 줄 수 있다

같은 그래프나 에이전트에도 invoke마다 다른 context를 넣을 수 있다.

모델 교체 실험, memory + `thread_id` 조합과도 잘 맞는다.
- 다른 모델, 다른 DB, 다른 권한
- A/B 테스트, 테넌트별 설정

### (5) 타입과 구조를 명시한다

`context_schema=MyContextSchema`로 **필요한 의존성 구조**를 코드에 드러낸다.

- IDE 자동완성
- `get_runtime(MyContextSchema)` 타입 힌트

### (6) 핵심 로직과 프레임워크 래퍼를 분리한다

| 계층 | 예시 | 역할 |
|------|------|------|
| 순수 함수 또는 서비스 | `execute_sql_core(query, db)` | context 없이 단위 테스트 |
| 노드 또는 `@tool` 래퍼 | `execute_sql` | context에서 의존성(예, `db`)을 꺼내 core 호출 |

이렇게 하면 **LangGraph 장점**(컨텍스트, 체크포인팅)과 **일반적인 설계**(테스트, 재사용)를 함께 가져간다.

- - -

## 4. 주의점

### (1) 그래프 밖에서 단독 호출하기 어렵다

`get_runtime()` 기반 패턴은 **컴파일된 그래프/에이전트 실행 안**을 전제로 한다.  
- 도구를 `invoke()`만으로 테스트하려면 런타임이 없어 실패할 수 있다.  
- 이때는 **순수 함수**나 **리소스를 직접 넘기는 테스트**를 쓴다.

```python
# (x) list_tables.invoke('')  # 런타임 없음 -> 오류
# (o) db.get_usable_table_names()  # 직접 테스트
# (o) execute_sql_core(query, db)  # 순수 함수 테스트
```

### (2) 의존성이 시그니처에 드러나지 않는다

함수 시그니처만 보면 DB나 LLM 등이 보이지 않을 수 있다.  
**호출부의 `context=...`** 까지 함께 봐야 의존성을 추적할 수 있다.

### (3) Memory와 혼동하기 쉽다

- **Runtime Context**: 매 실행마다 주입하며, 대화 기록에 보통 **저장하지 않는다.**
- **Memory (checkpointer)**: `thread_id`로 **messages/state를 지속**한다.

### (4) 스키마 이름과 실제 검증 범위가 다를 수 있다 (검증 착시)

Pydantic 등으로 context 스키마를 정의해도, **이미 만들어진 외부 객체**(DB 클라이언트, LLM 인스턴스)는 내용 검증 이점이 거의 없다.  
- 예) `db`, `model`은 **외부 객체 레퍼런스**이므로 Pydantic이 내용을 거의 검증하지 않는다.

스키마는 주로 **'어떤 필드가 있는지' 구조를 명시**하는 용도에 가깝다.
- 직렬화 가능한 값(`str`, `Literal`, 숫자) -> 검증 이점 큼
- 외부 객체 레퍼런스 -> `arbitrary_types_allowed` 등 설정 필요, 검증은 제한적
    - 예) Pydantic을 쓸 때는 `ConfigDict(arbitrary_types_allowed=True)`를 설정해야 한다.

### (5) 설정을 누락하면 런타임에 오류가 난다

- `context=`를 생략하면 주입이 되지 않는다.
- 선택 필드를 `None`으로 두거나 context에 아예 넣지 않으면, 노드·도구가 **방어 없이** 그 필드를 사용할 때 문제가 난다.
    - 예) `llm=None`인 상태에서 `llm.invoke(...)`를 바로 호출하는 경우

이때는 **실행 중**에야 오류가 드러나는 경우가 많다.

> **참고:** 노드·도구에 `if resource is None: return '...'` 같은 **방어 코드**를 넣으면, 선택 필드를 context에 넣지 않아도 **런타임 예외가 발생하지 않는다.** 필수 리소스가 없을 때 안내 메시지를 반환하도록 설계한 경우이며, 이는 **의도된 동작**으로 보면 된다.

### (6) 복잡도가 늘어난다

전역 변수나 단순 dataclass보다 개념과 보일러플레이트가 늘어난다.  
작은 일회성 스크립트에는 과할 수 있다.

- - -


## 5. 언제 특히 잘 맞는가?

다음 조건이면 Runtime Context를 쓰기 좋다.

- 여러 노드나 도구가 **같은 리소스**를 공유한다.
    - 예) 여러 `@tool`이 **같은 DB나 LLM**을 공유한다.
- 대화와 흐름은 **State / Memory**로 관리하고, 인프라는 **context**로 분리한다.
- `create_agent` 또는 `StateGraph`에서 **`context_schema` 패턴**을 일관되게 쓴다.

- - -

## 참고: LangGraph API에서의 연결

```python
# StateGraph 예시
graph = StateGraph(State, context_schema=MyContextSchema)
compiled = graph.compile()
compiled.invoke(input, context=MyContextSchema(...))

# create_agent 예시
agent = create_agent(model=..., tools=..., context_schema=MyContextSchema)
agent.invoke(input=..., context=MyContextSchema(...))
```

```python
agent = create_agent(
    model=model,
    tools=get_sql_tools(),
    context_schema=SqlAgentContextSchema,
)

result = agent.invoke(
    input={'messages': [HumanMessage(user_message)]},
    context=SqlAgentContextSchema(db=db, model=model),
)
```

- **`context_schema`**: 그래프가 기대하는 context **타입(구조)**를 정의한다.
- **`context`**: 이번 실행에 넣을 **실제 값 또 인스턴스**를 주입한다.

- - -

## 참고: Context 스키마 정의 방식

| 방식 | 특징 |
|------|------|
| **Pydantic `BaseModel`** | 강의·프로젝트에서 스키마 이름 통일, `Field`·기본값. 외부 객체는 `arbitrary_types_allowed` 등 필요 |
| **`@dataclass`** | 가볍고 단순. LangGraph 공식 예시에서 흔함 |
| **`TypedDict` / `dict`** | 유연하지만 타입 안정성·검증은 약함 |

선택한 방식과 관계없이, **`context_schema`와 `get_runtime(동일 타입)`을 일치**시키는 것이 중요하다.
