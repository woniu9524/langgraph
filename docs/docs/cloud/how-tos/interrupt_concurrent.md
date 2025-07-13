# 如何使用 interrupt 选项

本指南假定您已了解什么是双重文本（double-texting）。您可以在 [双重文本概念指南](../../concepts/double_text.md) 中了解相关知识。

本指南涵盖了双重文本的 `interrupt` 选项，该选项会中断图的先前运行，并使用双重文本启动新运行。此选项不会删除第一次运行，而是将其保留在数据库中，但将其状态设置为 `interrupted`。下面是一个使用 `interrupt` 选项的快速示例。

## 设置

首先，我们将定义一个快速的辅助函数来打印 JS 和 CURL 模型输出（如果您使用 Python，可以跳过此步骤）：

=== "Javascript"

    ```js
    function prettyPrint(m) {
      const padded = " " + m['type'] + " ";
      const sepLen = Math.floor((80 - padded.length) / 2);
      const sep = "=".repeat(sepLen);
      const secondSep = sep + (padded.length % 2 ? "=" : "");
      
      console.log(`${sep}${padded}${secondSep}`);
      console.log("\n\n");
      console.log(m.content);
    }
    ```

=== "CURL"

    ```bash
    # PLACE THIS IN A FILE CALLED pretty_print.sh
    pretty_print() {
      local type="$1"
      local content="$2"
      local padded=" $type "
      local total_width=80
      local sep_len=$(( (total_width - ${#padded}) / 2 ))
      local sep=$(printf '=%.0s' $(eval "echo {1.."${sep_len}"}"))
      local second_sep=$sep
      if (( (total_width - ${#padded}) % 2 )); then
        second_sep="${second_sep}="
      fi

      echo "${sep}${padded}${second_sep}"
      echo
      echo "$content"
    }
    ```

现在，让我们导入所需的包并实例化我们的客户端、助手和线程。

=== "Python"

    ```python
    import asyncio

    from langchain_core.messages import convert_to_messages
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // Using the graph deployed with the name "agent"
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

## 创建运行

现在我们可以启动我们的两个运行，并加入第二个运行直到它完成：

=== "Python"

    ```python
    # the first run will be interrupted
    interrupted_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    # sleep a bit to get partial outputs from the first run
    await asyncio.sleep(2)
    run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
        multitask_strategy="interrupt",
    )
    # wait until the second run completes
    await client.runs.join(thread["thread_id"], run["run_id"])
    ```

=== "Javascript"

    ```js
    // the first run will be interrupted
    let interruptedRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      { input: { messages: [{ role: "human", content: "what's the weather in sf?" }] } }
    );
    // sleep a bit to get partial outputs from the first run
    await new Promise(resolve => setTimeout(resolve, 2000)); 

    let run = await client.runs.create(
      thread["thread_id"],
      assistantId,
      { 
        input: { messages: [{ role: "human", content: "what's the weather in nyc?" }] },
        multitaskStrategy: "interrupt" 
      }
    );

    // wait until the second run completes
    await client.runs.join(thread["thread_id"], run["run_id"]);
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in sf?\"}]},
    }" && sleep 2 && curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in nyc?\"}]},
      \"multitask_strategy\": \"interrupt\"
    }" && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join
    ```

## 查看运行结果

我们可以看到线程中包含了来自第一次运行的部分数据以及来自第二次运行的数据。

=== "Python"

    ```python
    state = await client.threads.get_state(thread["thread_id"])

    for m in convert_to_messages(state["values"]["messages"]):
        m.pretty_print()
    ```

=== "Javascript"

    ```js
    const state = await client.threads.getState(thread["thread_id"]);

    for (const m of state['values']['messages']) {
      prettyPrint(m);
    }
    ```

=== "CURL"

    ```bash
    source pretty_print.sh && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \
    jq -c '.values.messages[]' | while read -r element; do
        type=$(echo "$element" | jq -r '.type')
        content=$(echo "$element" | jq -r '.content | if type == "array" then tostring else . end')
        pretty_print "$type" "$content"
    done
    ```

输出：

    ================================ Human Message =================================
    
    what's the weather in sf?
    ================================== Ai Message ==================================
    
    [{'id': 'toolu_01MjNtVJwEcpujRGrf3x6Pih', 'input': {'query': 'weather in san francisco'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    Tool Calls:
      tavily_search_results_json (toolu_01MjNtVJwEcpujRGrf3x6Pih)
     Call ID: toolu_01MjNtVJwEcpujRGrf3x6Pih
      Args:
        query: weather in san francisco
    ================================= Tool Message =================================
    Name: tavily_search_results_json
    
    [{"url": "https://www.wunderground.com/hourly/us/ca/san-francisco/KCASANFR2002/date/2024-6-18", "content": "High 64F. Winds W at 10 to 20 mph. A few clouds from time to time. Low 49F. Winds W at 10 to 20 mph. Temp. San Francisco Weather Forecasts. Weather Underground provides local & long-range weather ..."}]
    ================================ Human Message =================================
    
    what's the weather in nyc?
    ================================== Ai Message ==================================
    
    [{'id': 'toolu_01KtE1m1ifPLQAx4fQLyZL9Q', 'input': {'query': 'weather in new york city'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    Tool Calls:
      tavily_search_results_json (toolu_01KtE1m1ifPLQAx4fQLyZL9Q)
     Call ID: toolu_01KtE1m1ifPLQAx4fQLyZL9Q
      Args:
        query: weather in new york city
    ================================= Tool Message =================================
    Name: tavily_search_results_json
    
    [{"url": "https://www.accuweather.com/en/us/new-york/10021/june-weather/349727", "content": "Get the monthly weather forecast for New York, NY, including daily high/low, historical averages, to help you plan ahead."}]
    ================================== Ai Message ==================================
    
    The search results provide weather forecasts and information for New York City. Based on the top result from AccuWeather, here are some key details about the weather in NYC:
    
    - This is a monthly weather forecast for New York City for the month of June.
    - It includes daily high and low temperatures to help plan ahead.
    - Historical averages for June in NYC are also provided as a reference point.
    - More detailed daily or hourly forecasts with precipitation chances, humidity, wind, etc. can be found by visiting the AccuWeather page.
    
    So in summary, the search provides a convenient overview of the expected weather conditions in New York City over the next month to give you an idea of what to prepare for if traveling or making plans there. Let me know if you need any other details!


验证原始的、被中断的运行是否已被中断

=== "Python"

    ```python
    print((await client.runs.get(thread["thread_id"], interrupted_run["run_id"]))["status"])
    ```

=== "Javascript"

    ```js
    console.log((await client.runs.get(thread['thread_id'], interruptedRun["run_id"]))["status"])
    ```

输出：

    ```
    'interrupted'
    ```