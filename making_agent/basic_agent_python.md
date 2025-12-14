### 🔴 Can i build agent without any framework where i want to build an agent where there will be feature like rag and tool calling in python?

Yes, you can absolutely build an AI agent in Python from scratch without relying on any frameworks like LangChain or LlamaIndex. This involves using libraries such as the OpenAI API (or similar LLM providers), basic Python structures for tool definitions, prompt engineering to simulate tool calling, and custom retrieval mechanisms for RAG. The core idea is to create a loop where the agent reasons about user input, decides on actions (like calling tools), executes them, and generates responses, all managed in pure Python code.

### High-Level Steps to Build the Agent

1. **Set Up Your Environment and LLM Access**:
   - Install minimal dependencies like `openai` (for GPT models) or alternatives like `huggingface_hub` for open-source LLMs. You'll also need any API-specific libraries for tools (e.g., `wikipedia` for knowledge retrieval, `psycopg2` for databases).
   - Initialize an LLM client. For example:
     ```python
     import openai
     import os
     import json

     openai.api_key = os.getenv("OPENAI_API_KEY")
     client = openai.OpenAI()
     ```

2. **Define Tools**:
   - Create simple Python functions for each tool. Each should take input, perform an action, and return structured output (e.g., JSON string).
   - For tool calling, define a schema (as a list of dicts) describing each tool's name, description, and parameters. This helps the LLM understand when and how to use them.
   - Example tools:
     - A calculator tool for math operations.
     - A web search or Wikipedia tool for external knowledge (this ties into RAG).
     ```python
     def calculate_expression(expression: str) -> str:
         try:
             result = eval(expression)  # Add safety checks in production
             return json.dumps({"result": result})
         except Exception as e:
             return json.dumps({"error": str(e)})

     def search_wikipedia(query: str) -> str:
         import wikipedia
         try:
             summary = wikipedia.summary(query, sentences=3)
             return json.dumps({"summary": summary})
         except Exception as e:
             return json.dumps({"error": str(e)})

     tools = [
         {
             "type": "function",
             "function": {
                 "name": "calculate_expression",
                 "description": "Evaluate a mathematical expression.",
                 "parameters": {
                     "type": "object",
                     "properties": {"expression": {"type": "string"}},
                     "required": ["expression"]
                 }
             }
         },
         {
             "type": "function",
             "function": {
                 "name": "search_wikipedia",
                 "description": "Search Wikipedia for information.",
                 "parameters": {
                     "type": "object",
                     "properties": {"query": {"type": "string"}},
                     "required": ["query"]
                 }
             }
         }
     ]
     ```


3. **Implement Tool Calling**:
   - Use prompt engineering to make the LLM output a structured response (e.g., JSON indicating which tool to call and with what arguments). Alternatively, leverage the LLM's built-in tool-calling if available (like OpenAI's `tools` parameter in chat completions).
   - Parse the LLM's output in Python and execute the corresponding function.
   - Handle the response by feeding tool outputs back into the LLM for further reasoning (ReAct-style loop: Thought → Action → Observation).
   - Example execution function:
     ```python
     def execute_tool(tool_call):
         name = tool_call['function']['name']
         args = json.loads(tool_call['function']['arguments'])
         if name == "calculate_expression":
             return calculate_expression(args['expression'])
         elif name == "search_wikipedia":
             return search_wikipedia(args['query'])
         return json.dumps({"error": "Unknown tool"})
     ```


4. **Incorporate RAG (Retrieval-Augmented Generation)**:
   - For RAG, build a simple retrieval system: Use a dictionary, file, or database as your knowledge base. Implement a search function that retrieves relevant documents based on keywords, embeddings (via `numpy` or `sentence-transformers`), or vector similarity.
   - When the agent detects a need for external info (via tool calling), retrieve data and augment the LLM prompt with it before generating the final response.
   - Example: Extend the Wikipedia tool or add a custom knowledge base search. For a vector-based RAG, compute embeddings offline and query them at runtime without external frameworks.
   - In the agent loop, after retrieval, include the retrieved content in the conversation history for the LLM to use in generation.

5. **Create the Agent Loop**:
   - Build a class or function to manage the conversation: Maintain a list of messages (system prompt, user input, assistant responses, tool outputs).
   - In a loop (with a max iteration limit to prevent infinite cycles), call the LLM with the current messages and tools. If tool calls are returned, execute them and append results as "tool" role messages. Continue until no more tools are needed.
   - System prompt example: "You are an agent that can use tools to answer questions. Reason step by step and call tools when needed."
   - Full loop snippet:
     ```python
     def process_query(messages, max_iterations=5):
         iteration = 0
         while iteration < max_iterations:
             response = client.chat.completions.create(
                 model="gpt-4o",
                 messages=messages,
                 tools=tools,
                 tool_choice="auto"
             )
             msg = response.choices[0].message
             messages.append(msg)
             if not msg.tool_calls:
                 return msg.content
             for call in msg.tool_calls:
                 result = execute_tool(call)
                 messages.append({"role": "tool", "content": result, "tool_call_id": call.id})
             iteration += 1
         return "Max iterations reached."
     ```


6. **Add Memory and Enhancements**:
   - Store conversation history in a list or database for multi-turn interactions.
   - Summarize long histories to stay within token limits (use the LLM to generate summaries).
   - Handle errors gracefully, add logging, and test with various queries.

This approach gives you full control and is extensible—start simple and scale by adding more tools or advanced retrieval (e.g., integrating FAISS for vector search without frameworks). For full tutorials, check the linked resources for complete code examples.
