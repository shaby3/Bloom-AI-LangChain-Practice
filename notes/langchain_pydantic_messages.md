# LangChain 메시지와 Pydantic

## 핵심 요약

LangChain에서 출력되는 `HumanMessage`, `AIMessage`, `SystemMessage` 등은 모두 **Pydantic의 `BaseModel`을 상속**받고 있다. 따라서 Pydantic이 제공하는 기능(메소드)을 전부 사용할 수 있다.

---

## 상속 구조

```
pydantic.BaseModel
   └── langchain_core.load.Serializable
          └── langchain_core.messages.BaseMessage
                 ├── HumanMessage
                 ├── AIMessage
                 ├── SystemMessage
                 ├── ToolMessage
                 └── FunctionMessage
```

최상위에 `pydantic.BaseModel`이 있기 때문에, Pydantic 기능이 전부 상속된다.

---

## AIMessage 내부를 들여다보는 방법

`model.invoke()`의 반환값(`AIMessage`)은 딕셔너리가 아니라 Pydantic 모델 객체라서 `.keys()`는 동작하지 않는다. 대신 아래 메소드들을 쓴다.

```python
response = model.invoke("안녕하세요")

# 1. 모든 필드를 dict로 펼쳐서 보기
response.model_dump()

# 2. 필드 이름만 보기
response.model_fields.keys()
# 또는
list(response.model_dump().keys())

# 3. 예쁘게 JSON으로 보기
response.model_dump_json(indent=2)

# 4. 한글 깨짐 없이 보기
import json
print(json.dumps(response.model_dump(), indent=2, ensure_ascii=False, default=str))
```

---

## Pydantic v2 주요 메소드

| 메소드 | 설명 |
|---|---|
| `model_dump()` | 필드를 **dict**로 변환 |
| `model_dump_json()` | 필드를 **JSON 문자열**로 변환 |
| `model_fields` | 정의된 필드 메타정보 (클래스 속성) |
| `model_validate(data)` | dict → 모델 객체로 역변환 |
| `model_copy()` | 객체 복제 |

### Pydantic v1 → v2 변화

예전 v1에서는 `.dict()`, `.json()`, `.copy()`였는데, v2부터 **`model_` 접두사**가 붙었다. 사용자가 필드 이름으로 `dict`, `json` 같은 걸 쓰면 충돌이 나서, 프레임워크 메소드를 `model_*`로 격리한 것.

> `model_dump()`는 LangChain 기능이 아니라 **Pydantic이 모든 모델에 기본으로 달아주는 기능**이다.

---

## LangChain에서 Pydantic 모델인 것들

메시지 말고도 LangChain/LangGraph의 대부분 객체가 Pydantic 모델이다.

- **Messages**: `HumanMessage`, `AIMessage`, `SystemMessage`, `ToolMessage` ...
- **Outputs**: `ChatGeneration`, `LLMResult`, `AgentAction`, `AgentFinish`
- **Documents**: `Document` (RAG에서 쓰는 그 객체)
- **Tool 관련**: `ToolCall`, `BaseTool`의 `args_schema`
- **Structured Output**: `with_structured_output()`으로 만든 스키마

어떤 LangChain 객체를 받아도 "안에 뭐가 들었지?" 싶으면 거의 항상 아래 3가지가 통한다.

```python
obj.model_dump()               # 전체 필드 dict로
obj.model_fields.keys()        # 필드 이름만
obj.model_dump_json(indent=2)  # 예쁘게 JSON으로
```

---

## `model.invoke()` vs `agent.invoke()` 반환 타입 차이

| 호출 | 반환 타입 | `.keys()` | 접근 방법 |
|---|---|---|---|
| `model.invoke(...)` | `AIMessage` (Pydantic) | ❌ | `response.content`, `response.model_dump()` |
| `agent.invoke(...)` | `dict` (AgentState TypedDict) | ✅ | `response["messages"][-1].content` |

- **채팅 모델 직접 호출** → 단일 `AIMessage` 객체 반환
- **에이전트 그래프 실행** → 상태 딕셔너리(`{"messages": [...], ...}`) 반환

---

## 주의점

LangChain이 아직 Pydantic v1 호환 레이어도 함께 쓰고 있어서, 버전에 따라 드물게 v1 스타일(`.dict()`, `.json()`)이 남아있는 경우가 있다. 하지만 `langchain_core` 0.3+ 는 **Pydantic v2 기반**이라 `model_dump()`가 표준.

---

## 결론

> **LangChain 객체 ≈ Pydantic 모델 → `model_dump()`로 내부 들여다보기 가능**

이 공식이 거의 항상 성립한다.
