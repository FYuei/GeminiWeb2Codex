# Codex + gemini-web2api 直连配置手册 (供 Agent 调用)

本文件记录了将 Codex 客户端直接连接至本地 `gemini-web2api` 流式代理的完整配置参数与代码修改逻辑，以便其他 AI Agent 读取、调用和自动化配置。

---

## 1. 系统架构关系

整个链路为纯 API 直连模式，去除了不必要的中转层以提高稳定性：
```
Codex 客户端 (Responses 协议) 
      └── 直连 ──> http://127.0.0.1:8081/v1 (gemini-web2api 服务)
```

---

## 2. 核心配置文件

### 2.1 Codex 客户端配置文件 (`~/.codex/config.toml`)
必须配置 `wire_api = "responses"` 并将 `base_url` 设为 `8081`：
```toml
model_provider = "custom"
model = "gemini-3.5-flash-thinking"
notify = ["/Users/Yue/.codex/computer-use/Codex Computer Use.app/Contents/SharedSupport/SkyComputerUseClient.app/Contents/MacOS/SkyComputerUseClient", "turn-ended"]
model_reasoning_effort = "medium"

[model_providers]
[model_providers.custom]
name = "custom"
wire_api = "responses"
requires_openai_auth = true
base_url = "http://127.0.0.1:8081/v1"
experimental_bearer_token = "gemini"
```

### 2.2 Codex++ 会话配置文件 (`~/.codex-session-delete/settings.json`)
确保 `relayMode` 设为 `"pureApi"` 纯直连模式，防止启动时覆盖 `config.toml`：
```json
{
  "activeRelayId": "default",
  "relayProfiles": [
    {
      "id": "default",
      "name": "Geminiweb2API",
      "upstreamBaseUrl": "http://127.0.0.1:8081/v1",
      "protocol": "responses",
      "relayMode": "pureApi",
      "testModel": "gemini-3.5-flash-thinking",
      "configContents": "model = \"gemini-3.5-flash-thinking\"\nmodel_provider = \"custom\"\n\n[marketplaces.openai-curated]\nsource_type = \"local\"\nsource = '\\\\?\\/Users/yue/.codex/.tmp/plugins'\n\n[projects.\"/Users/Yue/Documents/ 语音校准\"]\ntrust_level = \"trusted\"\n\n[plugins.\"computer-use@openai-bundled\"]\nenabled = true\n\n[plugins.\"pdf@openai-primary-runtime\"]\nenabled = true\n\n[plugins.\"template-creator@openai-primary-runtime\"]\nenabled = true\n\n[model_providers]\n\n[model_providers.custom]\nname = \"custom\"\nwire_api = \"responses\"\nrequires_openai_auth = true\nbase_url = \"http://127.0.0.1:8081/v1\"\n"
    }
  ]
}
```

---

## 3. 服务端源码修改

为了让 Codex 前端 UI 能够正常初始化和流式增量显示气泡，`gemini-web2api` 的 `handle_responses` 接口必须吐出完整的 `added` 事件序列，并使用生成器进行真流式生成。

代码修改位于：`gemini_web2api.py` 的 `handle_responses` 方法中：

```python
        rid = f"resp_{uuid.uuid4().hex[:16]}"
        mid = f"msg_{uuid.uuid4().hex[:12]}"
        stream = req.get("stream", False)

        if stream and not tools:
            try:
                self.send_response(200)
                self.send_header("Content-Type", "text/event-stream")
                self.send_header("Cache-Control", "no-cache")
                self.send_header("Access-Control-Allow-Origin", "*")
                self.end_headers()

                # 1. response.created
                ev = {"type": "response.created", "response": {"id": rid, "object": "response", "status": "in_progress", "model": model_name, "output": []}}
                self.wfile.write(f"event: response.created\ndata: {json.dumps(ev)}\n\n".encode())
                self.wfile.flush()

                # 2. response.output_item.added (必须：用于告诉前端初始化 assistant 消息气泡)
                ev = {"type": "response.output_item.added", "output_index": 0, "item": {"id": mid, "type": "message", "status": "in_progress", "role": "assistant", "content": []}}
                self.wfile.write(f"event: response.output_item.added\ndata: {json.dumps(ev)}\n\n".encode())
                self.wfile.flush()

                # 3. response.content_part.added (必须：用于初始化气泡内的文本分区)
                ev = {"type": "response.content_part.added", "item_id": mid, "output_index": 0, "content_index": 0, "part": {"type": "output_text", "text": "", "annotations": []}}
                self.wfile.write(f"event: response.content_part.added\ndata: {json.dumps(ev)}\n\n".encode())
                self.wfile.flush()

                # Stream chunks
                full_text = ""
                for delta_text in gemini_stream_generate_iter(prompt, model_id, think_mode):
                    full_text += delta_text
                    ev = {"type": "response.output_text.delta", "item_id": mid, "output_index": 0, "content_index": 0, "delta": delta_text}
                    self.wfile.write(f"event: response.output_text.delta\ndata: {json.dumps(ev, ensure_ascii=False)}\n\n".encode())
                    self.wfile.flush()

                # 4. response.output_text.done
                ev = {"type": "response.output_text.done", "item_id": mid, "output_index": 0, "content_index": 0, "text": full_text}
                self.wfile.write(f"event: response.output_text.done\ndata: {json.dumps(ev, ensure_ascii=False)}\n\n".encode())
                self.wfile.flush()

                # 5. response.content_part.done
                ev = {"type": "response.content_part.done", "item_id": mid, "output_index": 0, "content_index": 0, "part": {"type": "output_text", "text": full_text, "annotations": []}}
                self.wfile.write(f"event: response.content_part.done\ndata: {json.dumps(ev, ensure_ascii=False)}\n\n".encode())
                self.wfile.flush()

                # 6. response.output_item.done
                ev = {"type": "response.output_item.done", "output_index": 0, "item": {"id": mid, "type": "message", "status": "completed", "role": "assistant", "content": [{"type": "output_text", "text": full_text, "annotations": []}]}}
                self.wfile.write(f"event: response.output_item.done\ndata: {json.dumps(ev, ensure_ascii=False)}\n\n".encode())
                self.wfile.flush()

                # 7. response.completed
                output_obj = [{"id": mid, "type": "message", "status": "completed", "role": "assistant", "content": [{"type": "output_text", "text": full_text, "annotations": []}]}]
                resp_obj = {"id": rid, "object": "response", "status": "completed", "model": model_name, "output": output_obj,
                            "usage": {"input_tokens": len(prompt)//4, "output_tokens": len(full_text)//4, "total_tokens": (len(prompt)+len(full_text))//4}}
                self.wfile.write(f"event: response.completed\ndata: {json.dumps({'type': 'response.completed', 'response': resp_obj}, ensure_ascii=False)}\n\n".encode())
                
                # 8. data: [DONE]
                self.wfile.write(b"data: [DONE]\n\n")
                self.wfile.flush()
            except (BrokenPipeError, ConnectionResetError):
                pass
            except Exception as e:
                log(f"Responses Stream error: {e}")
            return
```

---

## 4. 运行与验证指令

1. **后台启动本地 API 代理 (端口 8081)**：
   ```bash
   cd /Users/Yue/Documents/jimengapi/gemini-web2api && python3 gemini_web2api.py
   ```
2. **测试接口流式输出序列**：
   ```bash
   curl -s -N -X POST http://127.0.0.1:8081/v1/responses \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer gemini" \
     -d '{"model": "gemini-3.5-flash-thinking", "input": "hi", "stream": true}'
   ```
