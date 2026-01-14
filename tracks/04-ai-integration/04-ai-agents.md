# 04. AI Agents

[← Назад к списку тем](README.md)

---

## Agent Fundamentals

### What is an AI Agent?

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AI Agent Architecture                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Agent = LLM + Tools + Memory + Planning                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                         LLM (Brain)                          │   │
│  │    - Reasoning & decision making                             │   │
│  │    - Natural language understanding                          │   │
│  │    - Tool selection & orchestration                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│         ┌────────────────────┼────────────────────┐                │
│         │                    │                    │                │
│         ▼                    ▼                    ▼                │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐           │
│  │   Tools    │      │   Memory   │      │  Planning  │           │
│  │ - Search   │      │ - Short    │      │ - Goals    │           │
│  │ - Code     │      │ - Long     │      │ - Steps    │           │
│  │ - APIs     │      │ - Context  │      │ - Reflect  │           │
│  └────────────┘      └────────────┘      └────────────┘           │
│                                                                     │
│  Loop: Observe → Think → Act → Observe → ...                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### ReAct Pattern (Reasoning + Acting)

```python
REACT_PROMPT = """Answer the question using the following format:

Question: {question}

Thought: Think about what to do
Action: tool_name[tool_input]
Observation: result of the action
... (repeat Thought/Action/Observation as needed)
Thought: I now know the answer
Final Answer: the answer

Available tools:
- search[query]: Search the web
- calculator[expression]: Calculate math
- lookup[term]: Look up a definition

Question: {question}
"""

def react_agent(question: str, max_steps: int = 5) -> str:
    prompt = REACT_PROMPT.format(question=question)

    for step in range(max_steps):
        response = llm.generate(prompt)
        prompt += response

        if "Final Answer:" in response:
            return extract_final_answer(response)

        if "Action:" in response:
            action = parse_action(response)
            observation = execute_tool(action.tool, action.input)
            prompt += f"\nObservation: {observation}\n"

    return "Could not find answer within step limit"

def execute_tool(tool: str, input: str) -> str:
    tools = {
        "search": web_search,
        "calculator": safe_eval,
        "lookup": knowledge_base_lookup
    }
    return tools[tool](input)
```

### Basic Agent Implementation

```python
from openai import OpenAI

class SimpleAgent:
    def __init__(self, tools: list[dict]):
        self.client = OpenAI()
        self.tools = tools
        self.messages = []

    def run(self, user_input: str) -> str:
        self.messages.append({"role": "user", "content": user_input})

        while True:
            response = self.client.chat.completions.create(
                model="gpt-4",
                messages=self.messages,
                tools=self.tools,
                tool_choice="auto"
            )

            message = response.choices[0].message
            self.messages.append(message)

            # If no tool calls, return response
            if not message.tool_calls:
                return message.content

            # Execute tool calls
            for tool_call in message.tool_calls:
                result = self.execute_tool(
                    tool_call.function.name,
                    json.loads(tool_call.function.arguments)
                )

                self.messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": str(result)
                })

    def execute_tool(self, name: str, args: dict) -> any:
        # Tool implementations
        if name == "search":
            return web_search(args["query"])
        elif name == "calculator":
            return eval(args["expression"])
        # ... more tools
```

---

## Tool Use / Function Calling

### OpenAI Function Calling

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City name, e.g. 'San Francisco'"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature unit"
                    }
                },
                "required": ["location"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_database",
            "description": "Search the product database",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "category": {"type": "string"},
                    "max_results": {"type": "integer", "default": 10}
                },
                "required": ["query"]
            }
        }
    }
]

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "What's the weather in Tokyo?"}],
    tools=tools,
    tool_choice="auto"  # or "none", "required", {"type": "function", "function": {"name": "..."}}
)

# Handle tool call
if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    print(f"Calling: {tool_call.function.name}")
    print(f"Args: {tool_call.function.arguments}")
```

### Claude Tool Use

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "City name"
                }
            },
            "required": ["location"]
        }
    }
]

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Weather in Paris?"}]
)

# Check for tool use
for block in response.content:
    if block.type == "tool_use":
        print(f"Tool: {block.name}")
        print(f"Input: {block.input}")

        # Execute and continue conversation
        tool_result = execute_tool(block.name, block.input)

        # Send result back
        followup = client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1024,
            tools=tools,
            messages=[
                {"role": "user", "content": "Weather in Paris?"},
                {"role": "assistant", "content": response.content},
                {
                    "role": "user",
                    "content": [{
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(tool_result)
                    }]
                }
            ]
        )
```

---

## Agent Architectures

### Single Agent

```python
# One agent with multiple tools
class SingleAgent:
    """Simple agent that can use multiple tools."""

    def __init__(self):
        self.tools = [
            SearchTool(),
            CalculatorTool(),
            CodeExecutorTool(),
            DatabaseTool()
        ]

    def run(self, task: str) -> str:
        messages = [{"role": "user", "content": task}]

        while True:
            response = self.think(messages)

            if response.is_final:
                return response.content

            if response.tool_call:
                result = self.execute(response.tool_call)
                messages.append({"role": "tool", "content": result})
```

### Multi-Agent Systems

```python
class MultiAgentOrchestrator:
    """Coordinate multiple specialized agents."""

    def __init__(self):
        self.agents = {
            "researcher": ResearchAgent(),
            "coder": CodingAgent(),
            "reviewer": ReviewAgent(),
            "writer": WritingAgent()
        }
        self.router = RouterAgent()

    def run(self, task: str) -> str:
        # Router decides which agent(s) to use
        plan = self.router.plan(task)

        results = {}
        for step in plan.steps:
            agent = self.agents[step.agent]
            context = self.gather_context(step, results)
            results[step.id] = agent.run(step.task, context)

        return self.synthesize(results)

# Example: Research task
# 1. Researcher gathers information
# 2. Writer creates draft
# 3. Reviewer provides feedback
# 4. Writer revises
```

### Hierarchical Agents

```python
class ManagerAgent:
    """High-level planning and delegation."""

    def __init__(self):
        self.workers = {
            "data": DataAgent(),
            "analysis": AnalysisAgent(),
            "report": ReportAgent()
        }

    def run(self, goal: str) -> str:
        # Create high-level plan
        plan = self.create_plan(goal)

        for task in plan.tasks:
            # Delegate to worker
            worker = self.workers[task.type]
            result = worker.run(task.description)

            # Review result
            if not self.review(result, task):
                # Provide feedback and retry
                result = worker.run(task.description, feedback=self.feedback)

            plan.results[task.id] = result

        return self.compile_results(plan)
```

---

## Planning Strategies

### Plan-and-Execute

```python
def plan_and_execute(task: str) -> str:
    """Create plan first, then execute steps."""

    # Step 1: Create plan
    plan_prompt = f"""Create a step-by-step plan to accomplish:
{task}

Output as numbered list. Each step should be atomic and executable."""

    plan = llm.generate(plan_prompt)
    steps = parse_steps(plan)

    # Step 2: Execute each step
    results = []
    for i, step in enumerate(steps):
        exec_prompt = f"""Execute this step:
{step}

Previous results:
{format_results(results)}

Provide the result of this step."""

        result = llm.generate(exec_prompt)
        results.append({"step": step, "result": result})

    # Step 3: Synthesize
    synth_prompt = f"""Original task: {task}

Completed steps and results:
{format_results(results)}

Provide the final answer."""

    return llm.generate(synth_prompt)
```

### Reflexion (Self-Reflection)

```python
def reflexion_agent(task: str, max_attempts: int = 3) -> str:
    """Agent that learns from mistakes."""

    memory = []  # Store past attempts

    for attempt in range(max_attempts):
        # Generate response with memory of past attempts
        response = generate_with_memory(task, memory)

        # Evaluate response
        evaluation = evaluate_response(task, response)

        if evaluation.success:
            return response

        # Reflect on failure
        reflection = reflect_on_failure(
            task=task,
            response=response,
            evaluation=evaluation
        )

        memory.append({
            "attempt": attempt,
            "response": response,
            "evaluation": evaluation,
            "reflection": reflection
        })

    return f"Failed after {max_attempts} attempts. Last response: {response}"

def reflect_on_failure(task, response, evaluation):
    prompt = f"""Task: {task}
Your response: {response}
Evaluation: {evaluation.feedback}

What went wrong? How can you do better next time?"""

    return llm.generate(prompt)
```

---

## Memory Systems

### Conversation Memory

```python
class ConversationMemory:
    """Short-term memory for conversation context."""

    def __init__(self, max_tokens: int = 4000):
        self.messages = []
        self.max_tokens = max_tokens

    def add(self, role: str, content: str):
        self.messages.append({"role": role, "content": content})
        self.trim()

    def trim(self):
        """Keep messages within token limit."""
        while self.count_tokens() > self.max_tokens:
            # Remove oldest messages (keep system prompt)
            if self.messages[0]["role"] == "system":
                self.messages.pop(1)
            else:
                self.messages.pop(0)

    def get_context(self) -> list:
        return self.messages.copy()
```

### Long-Term Memory with RAG

```python
class LongTermMemory:
    """Persistent memory using vector store."""

    def __init__(self):
        self.vector_store = VectorStore()

    def store(self, content: str, metadata: dict = None):
        """Store information for later retrieval."""
        embedding = embed(content)
        self.vector_store.add(
            embedding=embedding,
            content=content,
            metadata={
                "timestamp": datetime.now().isoformat(),
                **(metadata or {})
            }
        )

    def recall(self, query: str, top_k: int = 5) -> list[str]:
        """Retrieve relevant memories."""
        query_embedding = embed(query)
        results = self.vector_store.search(query_embedding, top_k=top_k)
        return [r.content for r in results]

class AgentWithMemory:
    def __init__(self):
        self.short_term = ConversationMemory()
        self.long_term = LongTermMemory()

    def run(self, user_input: str) -> str:
        # Recall relevant long-term memories
        memories = self.long_term.recall(user_input)

        # Build context
        context = f"Relevant memories:\n{format_memories(memories)}\n\n"

        # Generate response
        response = self.generate(context + user_input)

        # Store interaction in long-term memory
        self.long_term.store(f"User: {user_input}\nAgent: {response}")

        return response
```

---

## На интервью

### Типичные вопросы

1. **What is an AI agent?**
   - LLM + tools + memory + planning
   - Can take actions, not just generate text
   - Autonomous goal-directed behavior

2. **ReAct pattern?**
   - Reasoning + Acting
   - Thought → Action → Observation loop
   - Interleaves reasoning with tool use

3. **Single vs multi-agent?**
   - Single: simpler, one LLM orchestrates
   - Multi: specialized agents, collaboration
   - Trade-off: complexity vs capability

4. **How to handle tool errors?**
   - Retry with different parameters
   - Fall back to alternative tools
   - Report error to user gracefully

5. **Agent memory types?**
   - Short-term: conversation context
   - Long-term: RAG-based retrieval
   - Episodic: specific interactions

6. **Challenges with agents?**
   - Reliability and consistency
   - Cost (many LLM calls)
   - Latency
   - Error propagation

---

[← Назад к списку тем](README.md)
