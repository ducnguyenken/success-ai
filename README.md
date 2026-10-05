# Hướng dẫn sử dụng OpenAI Agents SDK (Python)

[OpenAI Agents SDK](https://github.com/openai/openai-agents-python) là thư viện gọn nhẹ để xây dựng ứng dụng AI dạng agent. SDK xoay quanh một vài khái niệm cơ bản:

| Khái niệm | Ý nghĩa |
|---|---|
| **Agent** | Một LLM được cấu hình với `instructions`, `tools`, `handoffs`, `guardrails`… |
| **Runner** | Chạy vòng lặp agent: gọi model → gọi tool → lặp lại cho đến khi có kết quả cuối |
| **Tools** | Hàm Python (hoặc agent khác, MCP server) mà agent có thể gọi |
| **Handoffs** | Cho phép một agent chuyển cuộc hội thoại sang agent khác |
| **Guardrails** | Kiểm tra đầu vào/đầu ra để chặn yêu cầu không hợp lệ |
| **Sessions** | Tự động lưu lịch sử hội thoại giữa các lần chạy |
| **Tracing** | Ghi lại toàn bộ quá trình chạy để debug trên OpenAI Dashboard |

---

## 1. Cài đặt

Yêu cầu Python 3.9+.

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install openai-agents
```

Thiết lập API key:

```bash
export OPENAI_API_KEY="sk-..."   # Windows PowerShell: $env:OPENAI_API_KEY="sk-..."
```

---

## 2. Agent đầu tiên

```python
from agents import Agent, Runner

agent = Agent(
    name="Assistant",
    instructions="Bạn là trợ lý hữu ích, luôn trả lời bằng tiếng Việt.",
)

result = Runner.run_sync(agent, "Viết một bài thơ haiku về lập trình.")
print(result.final_output)
```

Ba cách chạy agent:

- `Runner.run_sync(agent, input)` – chạy đồng bộ, tiện cho script.
- `await Runner.run(agent, input)` – chạy bất đồng bộ (async).
- `Runner.run_streamed(agent, input)` – stream kết quả theo thời gian thực.

Chọn model cụ thể:

```python
agent = Agent(name="Assistant", instructions="...", model="gpt-4.1")
```

---

## 3. Thêm Tools (function tools)

Dùng decorator `@function_tool`. SDK tự sinh schema từ type hint và docstring.

```python
import asyncio
from agents import Agent, Runner, function_tool

@function_tool
def get_weather(city: str) -> str:
    """Lấy thời tiết hiện tại của một thành phố.

    Args:
        city: Tên thành phố.
    """
    # Thay bằng lời gọi API thật
    return f"Thời tiết ở {city}: nắng, 32°C."

agent = Agent(
    name="Weather Agent",
    instructions="Dùng tool để trả lời câu hỏi về thời tiết.",
    tools=[get_weather],
)

async def main():
    result = await Runner.run(agent, "Thời tiết ở Hà Nội thế nào?")
    print(result.final_output)

asyncio.run(main())
```

Ngoài function tool, SDK còn hỗ trợ các hosted tool của OpenAI như `WebSearchTool`, `FileSearchTool`, `CodeInterpreterTool`:

```python
from agents import Agent, WebSearchTool

agent = Agent(name="Researcher", tools=[WebSearchTool()])
```

---

## 4. Structured Output (kết quả có cấu trúc)

Truyền một Pydantic model vào `output_type` để nhận kết quả đúng định dạng.

```python
from pydantic import BaseModel
from agents import Agent, Runner

class CalendarEvent(BaseModel):
    title: str
    date: str
    participants: list[str]

agent = Agent(
    name="Calendar Extractor",
    instructions="Trích xuất sự kiện lịch từ văn bản.",
    output_type=CalendarEvent,
)

result = Runner.run_sync(agent, "Thứ 6 này An và Bình họp review dự án.")
event: CalendarEvent = result.final_output
print(event.title, event.participants)
```

---

## 5. Handoffs – chuyển giao giữa các agent

Một agent "phân loại" có thể chuyển yêu cầu sang agent chuyên trách.

```python
from agents import Agent, Runner

billing_agent = Agent(
    name="Billing Agent",
    handoff_description="Xử lý câu hỏi về hóa đơn, thanh toán.",
    instructions="Bạn hỗ trợ khách hàng về thanh toán.",
)

tech_agent = Agent(
    name="Tech Support Agent",
    handoff_description="Xử lý lỗi kỹ thuật, sự cố sản phẩm.",
    instructions="Bạn hỗ trợ khách hàng về kỹ thuật.",
)

triage_agent = Agent(
    name="Triage Agent",
    instructions="Xác định loại yêu cầu và chuyển cho agent phù hợp.",
    handoffs=[billing_agent, tech_agent],
)

result = Runner.run_sync(triage_agent, "Tôi bị trừ tiền 2 lần trong tháng này.")
print(result.last_agent.name)   # Billing Agent
print(result.final_output)
```

### Agent làm tool (agents-as-tools)

Khác với handoff (chuyển hẳn quyền điều khiển), bạn có thể gọi agent khác như một tool và giữ quyền điều khiển ở agent chính:

```python
translator = Agent(name="Translator", instructions="Dịch sang tiếng Anh.")

orchestrator = Agent(
    name="Orchestrator",
    instructions="Dùng tool dịch khi người dùng yêu cầu.",
    tools=[
        translator.as_tool(
            tool_name="translate_to_english",
            tool_description="Dịch văn bản sang tiếng Anh.",
        )
    ],
)
```

---

## 6. Guardrails

Guardrail chạy song song với agent và có thể dừng quá trình khi phát hiện vi phạm.

```python
from pydantic import BaseModel
from agents import (
    Agent, Runner, GuardrailFunctionOutput,
    InputGuardrailTripwireTriggered, input_guardrail,
)

class HomeworkCheck(BaseModel):
    is_homework: bool
    reasoning: str

checker = Agent(
    name="Homework Checker",
    instructions="Kiểm tra xem người dùng có nhờ làm bài tập về nhà không.",
    output_type=HomeworkCheck,
)

@input_guardrail
async def homework_guardrail(ctx, agent, input):
    result = await Runner.run(checker, input, context=ctx.context)
    output = result.final_output_as(HomeworkCheck)
    return GuardrailFunctionOutput(
        output_info=output,
        tripwire_triggered=output.is_homework,
    )

agent = Agent(
    name="Support Agent",
    instructions="Hỗ trợ khách hàng.",
    input_guardrails=[homework_guardrail],
)

try:
    Runner.run_sync(agent, "Giải giúp tôi bài toán 2x + 3 = 7")
except InputGuardrailTripwireTriggered:
    print("Yêu cầu bị guardrail chặn.")
```

Tương tự, dùng `@output_guardrail` và tham số `output_guardrails=[...]` để kiểm tra đầu ra.

---

## 7. Sessions – ghi nhớ hội thoại

Không cần tự quản lý `to_input_list()`; Session tự lưu lịch sử.

```python
from agents import Agent, Runner, SQLiteSession

agent = Agent(name="Assistant", instructions="Trả lời ngắn gọn.")
session = SQLiteSession("user_123", "conversations.db")

Runner.run_sync(agent, "Tên tôi là Đức.", session=session)
result = Runner.run_sync(agent, "Tôi tên là gì?", session=session)
print(result.final_output)   # "Bạn tên là Đức."
```

---

## 8. Streaming

```python
import asyncio
from openai.types.responses import ResponseTextDeltaEvent
from agents import Agent, Runner

agent = Agent(name="Storyteller", instructions="Kể chuyện ngắn.")

async def main():
    result = Runner.run_streamed(agent, "Kể một câu chuyện về con mèo.")
    async for event in result.stream_events():
        if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
            print(event.data.delta, end="", flush=True)

asyncio.run(main())
```

---

## 9. Context – truyền dữ liệu nội bộ cho tools

`context` là object của riêng bạn (user info, DB connection…), **không** gửi lên LLM, nhưng tools có thể đọc được.

```python
from dataclasses import dataclass
from agents import Agent, Runner, RunContextWrapper, function_tool

@dataclass
class UserInfo:
    user_id: str
    name: str

@function_tool
def greet(ctx: RunContextWrapper[UserInfo]) -> str:
    """Chào người dùng hiện tại."""
    return f"Xin chào {ctx.context.name}!"

agent = Agent[UserInfo](name="Greeter", tools=[greet])

result = Runner.run_sync(agent, "Chào tôi đi", context=UserInfo("u1", "Đức"))
print(result.final_output)
```

---

## 10. Kết nối MCP Server

```python
import asyncio
from agents import Agent, Runner
from agents.mcp import MCPServerStdio

async def main():
    async with MCPServerStdio(
        params={
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "./data"],
        }
    ) as fs_server:
        agent = Agent(
            name="File Assistant",
            instructions="Dùng các tool filesystem để trả lời.",
            mcp_servers=[fs_server],
        )
        result = await Runner.run(agent, "Liệt kê các file trong thư mục.")
        print(result.final_output)

asyncio.run(main())
```

---

## 11. Tracing

Tracing được bật mặc định; xem tại <https://platform.openai.com/traces>. Gom nhiều lần chạy vào một trace:

```python
from agents import Agent, Runner, trace

agent = Agent(name="Joker", instructions="Kể chuyện cười.")

with trace("Joke workflow"):
    first = Runner.run_sync(agent, "Kể một chuyện cười.")
    second = Runner.run_sync(agent, f"Đánh giá chuyện cười này: {first.final_output}")
```

Tắt tracing: đặt biến môi trường `OPENAI_AGENTS_DISABLE_TRACING=1`.

---

## 12. Dùng model không phải OpenAI

Qua LiteLLM (`pip install "openai-agents[litellm]"`):

```python
from agents import Agent
from agents.extensions.models.litellm_model import LitellmModel

agent = Agent(
    name="Assistant",
    model=LitellmModel(model="anthropic/claude-sonnet-5-5", api_key="..."),
)
```

---

## 13. Mẹo & lưu ý

- Viết `instructions` rõ ràng, nêu khi nào nên dùng tool nào.
- Docstring của `@function_tool` chính là mô tả tool gửi cho model – hãy viết kỹ.
- Đặt `max_turns` trong `Runner.run(..., max_turns=10)` để tránh vòng lặp vô hạn.
- Bắt các exception: `MaxTurnsExceeded`, `InputGuardrailTripwireTriggered`, `OutputGuardrailTripwireTriggered`.
- Không commit API key – dùng biến môi trường hoặc file `.env` (nhớ thêm `.env` vào `.gitignore`).

## Tài liệu tham khảo

- Tài liệu chính thức: <https://openai.github.io/openai-agents-python/>
- Mã nguồn & ví dụ: <https://github.com/openai/openai-agents-python/tree/main/examples>
