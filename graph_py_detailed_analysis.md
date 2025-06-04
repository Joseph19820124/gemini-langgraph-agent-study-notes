# LangGraph智能搜索代理詳細代碼分析 - graph.py

## 概述
這個文件實現了一個基於LangGraph和Google Gemini模型的智能搜索代理。該代理能夠根據用戶問題自動生成搜索查詢、執行網絡搜索、反思研究結果，並最終提供綜合性答案。

## 詳細代碼分析

### 導入模塊部分

```python
import os
```
- **作用**: 導入Python標準庫的os模塊
- **用途**: 用於訪問操作系統相關功能，特別是環境變量的讀取

```python
from agent.tools_and_schemas import SearchQueryList, Reflection
```
- **作用**: 從自定義的tools_and_schemas模塊導入數據結構
- **SearchQueryList**: 可能是用於存儲搜索查詢列表的數據結構
- **Reflection**: 用於反思階段的數據結構，包含知識差距分析

```python
from dotenv import load_dotenv
```
- **作用**: 導入dotenv庫的load_dotenv函數
- **用途**: 從.env文件加載環境變量，便於管理API密鑰等敏感信息

```python
from langchain_core.messages import AIMessage
```
- **作用**: 導入LangChain核心的AIMessage類
- **用途**: 用於表示AI助手的消息，在最終結果中使用

```python
from langgraph.types import Send
```
- **作用**: 導入LangGraph的Send類型
- **用途**: 用於在圖中發送消息到特定節點，支持並行處理

```python
from langgraph.graph import StateGraph
```
- **作用**: 導入LangGraph的StateGraph類
- **用途**: 核心圖結構，用於定義狀態機和節點流程

```python
from langgraph.graph import START, END
```
- **作用**: 導入LangGraph的特殊節點標識符
- **START**: 圖的開始節點
- **END**: 圖的結束節點

```python
from langchain_core.runnables import RunnableConfig
```
- **作用**: 導入LangChain的配置類
- **用途**: 用於傳遞運行時配置參數

```python
from google.genai import Client
```
- **作用**: 導入Google GenAI客戶端
- **用途**: 直接與Google的生成式AI API交互

```python
from agent.state import (
    OverallState,
    QueryGenerationState,
    ReflectionState,
    WebSearchState,
)
```
- **作用**: 導入各種狀態定義
- **OverallState**: 整體狀態，包含所有必要的狀態信息
- **QueryGenerationState**: 查詢生成階段的狀態
- **ReflectionState**: 反思階段的狀態
- **WebSearchState**: 網絡搜索階段的狀態

```python
from agent.configuration import Configuration
```
- **作用**: 導入配置類
- **用途**: 管理代理的各種配置參數，如模型選擇、循環次數等

```python
from agent.prompts import (
    get_current_date,
    query_writer_instructions,
    web_searcher_instructions,
    reflection_instructions,
    answer_instructions,
)
```
- **作用**: 導入提示模板和日期函數
- **get_current_date**: 獲取當前日期
- **query_writer_instructions**: 查詢生成的指令模板
- **web_searcher_instructions**: 網絡搜索的指令模板
- **reflection_instructions**: 反思階段的指令模板
- **answer_instructions**: 最終答案生成的指令模板

```python
from langchain_google_genai import ChatGoogleGenerativeAI
```
- **作用**: 導入LangChain的Google Gemini聊天模型包裝器
- **用途**: 提供與Gemini模型交互的統一接口

```python
from agent.utils import (
    get_citations,
    get_research_topic,
    insert_citation_markers,
    resolve_urls,
)
```
- **作用**: 導入工具函數
- **get_citations**: 從響應中提取引用信息
- **get_research_topic**: 從消息中提取研究主題
- **insert_citation_markers**: 在文本中插入引用標記
- **resolve_urls**: 解析和簡化URL

### 環境配置部分

```python
load_dotenv()
```
- **作用**: 加載.env文件中的環境變量
- **目的**: 將敏感配置從代碼中分離出來

```python
if os.getenv("GEMINI_API_KEY") is None:
    raise ValueError("GEMINI_API_KEY is not set")
```
- **作用**: 檢查Gemini API密鑰是否已設置
- **安全性**: 確保必要的API密鑰存在，否則拋出異常
- **最佳實踐**: 早期失敗，避免運行時錯誤

```python
# Used for Google Search API
genai_client = Client(api_key=os.getenv("GEMINI_API_KEY"))
```
- **作用**: 初始化Google GenAI客戶端
- **用途**: 用於直接調用Google Search API和生成式AI功能
- **註釋**: 明確說明此客戶端用於Google Search API

## 核心節點函數

### 1. generate_query 函數

```python
def generate_query(state: OverallState, config: RunnableConfig) -> QueryGenerationState:
    """LangGraph node that generates a search queries based on the User's question.

    Uses Gemini 2.0 Flash to create an optimized search query for web research based on
    the User's question.

    Args:
        state: Current graph state containing the User's question
        config: Configuration for the runnable, including LLM provider settings

    Returns:
        Dictionary with state update, including search_query key containing the generated query
    """
```
- **作用**: 生成搜索查詢的節點函數
- **輸入**: 包含用戶問題的狀態和運行配置
- **輸出**: 包含生成的搜索查詢列表的狀態更新
- **模型**: 使用Gemini 2.0 Flash進行查詢優化

```python
    configurable = Configuration.from_runnable_config(config)
```
- **作用**: 從運行配置中提取自定義配置
- **目的**: 獲取模型選擇、參數設置等配置信息

```python
    # check for custom initial search query count
    if state.get("initial_search_query_count") is None:
        state["initial_search_query_count"] = configurable.number_of_initial_queries
```
- **作用**: 設置初始搜索查詢數量
- **邏輯**: 如果狀態中沒有設置，則使用配置中的默認值
- **靈活性**: 允許動態調整查詢數量

```python
    # init Gemini 2.0 Flash
    llm = ChatGoogleGenerativeAI(
        model=configurable.query_generator_model,
        temperature=1.0,
        max_retries=2,
        api_key=os.getenv("GEMINI_API_KEY"),
    )
```
- **作用**: 初始化Gemini語言模型
- **temperature=1.0**: 設置較高的創造性，生成多樣化的查詢
- **max_retries=2**: 設置最大重試次數，提高可靠性
- **模型選擇**: 使用配置中指定的查詢生成模型

```python
    structured_llm = llm.with_structured_output(SearchQueryList)
```
- **作用**: 為語言模型添加結構化輸出約束
- **目的**: 確保輸出符合SearchQueryList的格式要求
- **優勢**: 避免解析錯誤，提高輸出的可靠性

```python
    # Format the prompt
    current_date = get_current_date()
    formatted_prompt = query_writer_instructions.format(
        current_date=current_date,
        research_topic=get_research_topic(state["messages"]),
        number_queries=state["initial_search_query_count"],
    )
```
- **作用**: 格式化提示模板
- **current_date**: 提供時間上下文，有助於生成時效性查詢
- **research_topic**: 從用戶消息中提取核心研究主題
- **number_queries**: 指定需要生成的查詢數量

```python
    # Generate the search queries
    result = structured_llm.invoke(formatted_prompt)
    return {"query_list": result.query}
```
- **作用**: 調用模型生成查詢並返回結果
- **返回格式**: 包含查詢列表的字典，用於狀態更新

### 2. continue_to_web_research 函數

```python
def continue_to_web_research(state: QueryGenerationState):
    """LangGraph node that sends the search queries to the web research node.

    This is used to spawn n number of web research nodes, one for each search query.
    """
    return [
        Send("web_research", {"search_query": search_query, "id": int(idx)})
        for idx, search_query in enumerate(state["query_list"])
    ]
```
- **作用**: 並行分發搜索查詢到網絡研究節點
- **Send對象**: 每個Send創建一個到"web_research"節點的消息
- **並行處理**: 為每個查詢創建獨立的處理分支
- **ID分配**: 為每個查詢分配唯一ID，便於追蹤和結果匯總

### 3. web_research 函數

```python
def web_research(state: WebSearchState, config: RunnableConfig) -> OverallState:
    """LangGraph node that performs web research using the native Google Search API tool.

    Executes a web search using the native Google Search API tool in combination with Gemini 2.0 Flash.

    Args:
        state: Current graph state containing the search query and research loop count
        config: Configuration for the runnable, including search API settings

    Returns:
        Dictionary with state update, including sources_gathered, research_loop_count, and web_research_results
    """
```
- **作用**: 執行實際的網絡搜索操作
- **工具**: 使用Google Search API和Gemini 2.0 Flash
- **輸入**: 包含搜索查詢的狀態
- **輸出**: 包含搜索結果和來源信息的狀態更新

```python
    # Configure
    configurable = Configuration.from_runnable_config(config)
    formatted_prompt = web_searcher_instructions.format(
        current_date=get_current_date(),
        research_topic=state["search_query"],
    )
```
- **作用**: 配置搜索參數和格式化搜索提示
- **時間上下文**: 包含當前日期信息
- **研究主題**: 使用當前的搜索查詢作為研究主題

```python
    # Uses the google genai client as the langchain client doesn't return grounding metadata
    response = genai_client.models.generate_content(
        model=configurable.query_generator_model,
        contents=formatted_prompt,
        config={
            "tools": [{"google_search": {}}],
            "temperature": 0,
        },
    )
```
- **作用**: 使用Google GenAI客戶端執行搜索
- **工具配置**: 啟用google_search工具
- **temperature=0**: 設置為0以獲得確定性的搜索結果
- **為什麼不用LangChain**: 註釋說明LangChain客戶端不返回grounding metadata

```python
    # resolve the urls to short urls for saving tokens and time
    resolved_urls = resolve_urls(
        response.candidates[0].grounding_metadata.grounding_chunks, state["id"]
    )
```
- **作用**: 解析和簡化URL
- **優化目的**: 節省token數量和處理時間
- **grounding_metadata**: 使用Google API返回的基礎元數據

```python
    # Gets the citations and adds them to the generated text
    citations = get_citations(response, resolved_urls)
    modified_text = insert_citation_markers(response.text, citations)
    sources_gathered = [item for citation in citations for item in citation["segments"]]
```
- **作用**: 處理引用信息
- **citations**: 提取引用數據
- **modified_text**: 在文本中插入引用標記
- **sources_gathered**: 收集所有來源信息，用於後續處理

```python
    return {
        "sources_gathered": sources_gathered,
        "search_query": [state["search_query"]],
        "web_research_result": [modified_text],
    }
```
- **作用**: 返回搜索結果的狀態更新
- **列表格式**: 使用列表格式便於與其他結果合併

### 4. reflection 函數

```python
def reflection(state: OverallState, config: RunnableConfig) -> ReflectionState:
    """LangGraph node that identifies knowledge gaps and generates potential follow-up queries.

    Analyzes the current summary to identify areas for further research and generates
    potential follow-up queries. Uses structured output to extract
    the follow-up query in JSON format.

    Args:
        state: Current graph state containing the running summary and research topic
        config: Configuration for the runnable, including LLM provider settings

    Returns:
        Dictionary with state update, including search_query key containing the generated follow-up query
    """
```
- **作用**: 反思節點，分析知識差距並生成後續查詢
- **核心功能**: 識別研究不足的領域
- **輸出**: 結構化的後續查詢建議

```python
    configurable = Configuration.from_runnable_config(config)
    # Increment the research loop count and get the reasoning model
    state["research_loop_count"] = state.get("research_loop_count", 0) + 1
    reasoning_model = state.get("reasoning_model") or configurable.reasoning_model
```
- **作用**: 設置配置和追蹤研究循環
- **循環計數**: 增加研究循環計數器
- **模型選擇**: 優先使用狀態中的模型，否則使用配置默認值

```python
    # Format the prompt
    current_date = get_current_date()
    formatted_prompt = reflection_instructions.format(
        current_date=current_date,
        research_topic=get_research_topic(state["messages"]),
        summaries="\n\n---\n\n".join(state["web_research_result"]),
    )
```
- **作用**: 格式化反思提示
- **summaries**: 將所有搜索結果用分隔符連接
- **分隔符**: 使用"---"清晰分隔不同的搜索結果

```python
    # init Reasoning Model
    llm = ChatGoogleGenerativeAI(
        model=reasoning_model,
        temperature=1.0,
        max_retries=2,
        api_key=os.getenv("GEMINI_API_KEY"),
    )
    result = llm.with_structured_output(Reflection).invoke(formatted_prompt)
```
- **作用**: 初始化推理模型並執行反思
- **結構化輸出**: 確保返回符合Reflection格式的結果
- **temperature=1.0**: 允許創造性思考以發現知識差距

```python
    return {
        "is_sufficient": result.is_sufficient,
        "knowledge_gap": result.knowledge_gap,
        "follow_up_queries": result.follow_up_queries,
        "research_loop_count": state["research_loop_count"],
        "number_of_ran_queries": len(state["search_query"]),
    }
```
- **作用**: 返回反思結果
- **is_sufficient**: 判斷當前信息是否充足
- **knowledge_gap**: 識別出的知識差距
- **follow_up_queries**: 建議的後續查詢
- **統計信息**: 追蹤循環次數和已執行查詢數量

### 5. evaluate_research 函數

```python
def evaluate_research(
    state: ReflectionState,
    config: RunnableConfig,
) -> OverallState:
    """LangGraph routing function that determines the next step in the research flow.

    Controls the research loop by deciding whether to continue gathering information
    or to finalize the summary based on the configured maximum number of research loops.

    Args:
        state: Current graph state containing the research loop count
        config: Configuration for the runnable, including max_research_loops setting

    Returns:
        String literal indicating the next node to visit ("web_research" or "finalize_summary")
    """
```
- **作用**: 路由函數，決定研究流程的下一步
- **控制邏輯**: 基於信息充足性和循環次數決定是否繼續
- **返回值**: 字符串或Send對象列表

```python
    configurable = Configuration.from_runnable_config(config)
    max_research_loops = (
        state.get("max_research_loops")
        if state.get("max_research_loops") is not None
        else configurable.max_research_loops
    )
```
- **作用**: 獲取最大研究循環次數
- **優先級**: 狀態中的設置優先於配置中的默認值
- **防止無限循環**: 確保研究過程有明確的終止條件

```python
    if state["is_sufficient"] or state["research_loop_count"] >= max_research_loops:
        return "finalize_answer"
    else:
        return [
            Send(
                "web_research",
                {
                    "search_query": follow_up_query,
                    "id": state["number_of_ran_queries"] + int(idx),
                },
            )
            for idx, follow_up_query in enumerate(state["follow_up_queries"])
        ]
```
- **作用**: 決定流程分支
- **終止條件**: 信息充足或達到最大循環次數
- **繼續條件**: 為每個後續查詢創建新的搜索任務
- **ID管理**: 確保每個新查詢都有唯一ID

### 6. finalize_answer 函數

```python
def finalize_answer(state: OverallState, config: RunnableConfig):
    """LangGraph node that finalizes the research summary.

    Prepares the final output by deduplicating and formatting sources, then
    combining them with the running summary to create a well-structured
    research report with proper citations.

    Args:
        state: Current graph state containing the running summary and sources gathered

    Returns:
        Dictionary with state update, including running_summary key containing the formatted final summary with sources
    """
```
- **作用**: 最終化答案的節點
- **功能**: 去重來源、格式化結果、生成最終報告
- **輸出**: 包含引用的結構化研究報告

```python
    configurable = Configuration.from_runnable_config(config)
    reasoning_model = state.get("reasoning_model") or configurable.reasoning_model
```
- **作用**: 獲取推理模型配置
- **模型選擇**: 優先使用狀態中指定的模型

```python
    # Format the prompt
    current_date = get_current_date()
    formatted_prompt = answer_instructions.format(
        current_date=current_date,
        research_topic=get_research_topic(state["messages"]),
        summaries="\n---\n\n".join(state["web_research_result"]),
    )
```
- **作用**: 格式化最終答案生成提示
- **內容整合**: 將所有搜索結果整合到一個提示中

```python
    # init Reasoning Model, default to Gemini 2.5 Flash
    llm = ChatGoogleGenerativeAI(
        model=reasoning_model,
        temperature=0,
        max_retries=2,
        api_key=os.getenv("GEMINI_API_KEY"),
    )
    result = llm.invoke(formatted_prompt)
```
- **作用**: 初始化模型並生成最終答案
- **temperature=0**: 設置為0以獲得一致性的最終答案
- **模型註釋**: 默認使用Gemini 2.5 Flash

```python
    # Replace the short urls with the original urls and add all used urls to the sources_gathered
    unique_sources = []
    for source in state["sources_gathered"]:
        if source["short_url"] in result.content:
            result.content = result.content.replace(
                source["short_url"], source["value"]
            )
            unique_sources.append(source)
```
- **作用**: 處理URL引用和去重
- **URL替換**: 將短URL替換回原始URL
- **來源過濾**: 只保留實際使用的來源

```python
    return {
        "messages": [AIMessage(content=result.content)],
        "sources_gathered": unique_sources,
    }
```
- **作用**: 返回最終結果
- **AIMessage**: 將結果包裝為AI消息格式
- **來源列表**: 返回去重後的來源列表

## 圖結構構建

### StateGraph初始化

```python
# Create our Agent Graph
builder = StateGraph(OverallState, config_schema=Configuration)
```
- **作用**: 創建狀態圖構建器
- **OverallState**: 指定圖的整體狀態類型
- **config_schema**: 指定配置架構

### 節點添加

```python
# Define the nodes we will cycle between
builder.add_node("generate_query", generate_query)
builder.add_node("web_research", web_research)
builder.add_node("reflection", reflection)
builder.add_node("finalize_answer", finalize_answer)
```
- **作用**: 添加所有節點到圖中
- **節點映射**: 將字符串名稱映射到具體的函數

### 邊和流程控制

```python
# Set the entrypoint as `generate_query`
# This means that this node is the first one called
builder.add_edge(START, "generate_query")
```
- **作用**: 設置圖的入口點
- **START**: LangGraph的特殊起始節點
- **流程起點**: 所有執行都從查詢生成開始

```python
# Add conditional edge to continue with search queries in a parallel branch
builder.add_conditional_edges(
    "generate_query", continue_to_web_research, ["web_research"]
)
```
- **作用**: 添加條件邊實現並行搜索
- **並行分支**: 一個查詢生成多個並行的搜索任務

```python
# Reflect on the web research
builder.add_edge("web_research", "reflection")
```
- **作用**: 搜索完成後進入反思階段
- **聚合點**: 所有並行搜索結果在此匯聚

```python
# Evaluate the research
builder.add_conditional_edges(
    "reflection", evaluate_research, ["web_research", "finalize_answer"]
)
```
- **作用**: 基於反思結果決定下一步
- **分支邏輯**: 可能繼續搜索或進入最終化階段

```python
# Finalize the answer
builder.add_edge("finalize_answer", END)
```
- **作用**: 最終答案連接到結束節點
- **流程終點**: 圖執行的結束

### 圖編譯

```python
graph = builder.compile(name="pro-search-agent")
```
- **作用**: 編譯圖為可執行的實例
- **命名**: 為圖指定名稱"pro-search-agent"
- **優化**: 編譯過程可能包含圖結構優化

## 整體架構總結

這個智能搜索代理實現了一個sophisticated的多階段研究流程：

1. **查詢生成階段**: 將用戶問題轉換為優化的搜索查詢
2. **並行搜索階段**: 同時執行多個搜索查詢，提高效率
3. **反思階段**: 分析搜索結果，識別知識差距
4. **循環控制**: 基於信息充足性決定是否繼續搜索
5. **最終化階段**: 整合所有信息，生成綜合性答案

### 關鍵設計特點

- **並行處理**: 使用Send機制實現並行搜索，提高效率
- **循環控制**: 通過反思和評估實現智能的研究循環
- **引用管理**: 完整的URL處理和引用追蹤系統
- **配置靈活性**: 支持運行時配置調整
- **錯誤處理**: 包含重試機制和環境檢查
- **模塊化設計**: 清晰的職責分離和可復用組件

這個實現展示了如何使用LangGraph構建複雜的AI代理工作流，結合了現代語言模型的能力和傳統的搜索技術，創造了一個智能且高效的研究助手。