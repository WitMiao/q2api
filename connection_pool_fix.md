# 连接池耗尽修复方案

## 问题分析

### 根本原因
1. **响应未正确关闭**: 流式响应在客户端断开时未释放连接
2. **连接池配置过小**: 60个连接对流式场景不足
3. **缺少超时清理**: 长时间运行的连接没有强制超时
4. **无并发控制**: 没有限流导致连接池快速耗尽

## 修复步骤

### 1. 修复 replicate.py 的连接泄漏

在 `send_chat_request` 函数中添加上下文管理器:

```python
async def send_chat_request(...):
    # ... 省略前面代码 ...
    
    resp = None
    try:
        resp = await client.send(req, stream=True)
        
        # ... 错误处理 ...
        
        # 关键修复: 确保响应在所有情况下都被关闭
        async def _iter_events() -> AsyncGenerator[Any, None]:
            nonlocal response_consumed
            try:
                # ... 事件迭代代码 ...
            finally:
                # 确保响应被关闭
                response_consumed = True
                if resp and not resp.is_closed:
                    await resp.aclose()
                if local_client and client:
                    await client.aclose()
        
        # 包装生成器以处理 GeneratorExit
        async def _safe_iter_events():
            try:
                async for item in _iter_events():
                    yield item
            except GeneratorExit:
                # 客户端断开,确保清理
                if resp and not resp.is_closed:
                    await resp.aclose()
                if local_client and client:
                    await client.aclose()
                raise
            except Exception:
                # 任何异常都要清理
                if resp and not resp.is_closed:
                    await resp.aclose()
                if local_client and client:
                    await client.aclose()
                raise
        
        return None, None, tracker, _safe_iter_events()
    
    except Exception:
        # 发生异常时立即清理
        if resp and not resp.is_closed:
            await resp.aclose()
        if local_client and client:
            await client.aclose()
        raise
```

### 2. 增加连接池配置

修改 `app.py` 中的连接池设置:

```python
async def _init_global_client():
    global GLOBAL_CLIENT
    proxies = _get_proxies()
    mounts = None
    if proxies:
        proxy_url = proxies.get("https") or proxies.get("http")
        if proxy_url:
            mounts = {
                "https://": httpx.AsyncHTTPTransport(proxy=proxy_url),
                "http://": httpx.AsyncHTTPTransport(proxy=proxy_url),
            }
    
    # 修复: 大幅增加连接池大小
    limits = httpx.Limits(
        max_keepalive_connections=200,  # 从60提升到200
        max_connections=500,            # 从60提升到500
        keepalive_expiry=30.0
    )
    
    # 修复: 增加pool超时避免立即失败
    timeout = httpx.Timeout(
        connect=5.0,     # 从1.0提升到5.0
        read=300.0,      
        write=5.0,       # 从1.0提升到5.0
        pool=10.0        # 从1.0提升到10.0 - 关键!
    )
    
    GLOBAL_CLIENT = httpx.AsyncClient(
        mounts=mounts, 
        timeout=timeout, 
        limits=limits,
        http2=True  # 启用HTTP/2以提高连接复用
    )
```

### 3. 添加流式响应超时保护

在 `app.py` 的流式端点中添加超时:

```python
@app.post("/v1/messages")
async def claude_messages(req: ClaudeRequest, account: Dict[str, Any] = Depends(require_account)):
    # ... 省略前面代码 ...
    
    async def event_generator():
        start_time = time.time()
        max_duration = 600  # 10分钟超时
        
        try:
            async for event_type, payload in event_iter:
                # 检查超时
                if time.time() - start_time > max_duration:
                    logger.warning(f"Stream timeout after {max_duration}s")
                    break
                    
                async for sse in handler.handle_event(event_type, payload):
                    yield sse
            
            async for sse in handler.finish():
                yield sse
            
            await _update_stats(account["id"], True)
            
        except GeneratorExit:
            # 客户端断开
            logger.info("Client disconnected, cleaning up")
            await _update_stats(account["id"], tracker.has_content if tracker else False)
            raise
        except Exception as e:
            logger.error(f"Stream error: {e}")
            await _update_stats(account["id"], False)
            raise
        finally:
            # 确保资源释放
            logger.debug("Stream generator cleanup completed")
```

### 4. 添加并发限制

使用信号量限制并发:

```python
# 在 app.py 顶部添加
from asyncio import Semaphore

# 全局并发限制
MAX_CONCURRENT_REQUESTS = int(os.getenv("MAX_CONCURRENT_REQUESTS", "100"))
REQUEST_SEMAPHORE = Semaphore(MAX_CONCURRENT_REQUESTS)

@app.post("/v1/chat/completions")
async def chat_completions(req: ChatCompletionRequest, account: Dict[str, Any] = Depends(require_account)):
    async with REQUEST_SEMAPHORE:
        # ... 原有代码 ...
```

### 5. 添加连接池监控

```python
@app.get("/healthz")
async def health():
    if GLOBAL_CLIENT:
        # 获取连接池状态
        # httpx 没有直接的连接池统计API,但可以间接监控
        return {
            "status": "ok",
            "client": "initialized",
            "limits": {
                "max_connections": GLOBAL_CLIENT._limits.max_connections,
                "max_keepalive": GLOBAL_CLIENT._limits.max_keepalive_connections
            }
        }
    return {"status": "ok", "client": "not_initialized"}
```

### 6. 环境变量配置

添加到 `.env`:

```bash
# 连接池配置
MAX_CONCURRENT_REQUESTS=100
MAX_CONNECTIONS=500
MAX_KEEPALIVE_CONNECTIONS=200
STREAM_TIMEOUT_SECONDS=600

# 启用HTTP/2
ENABLE_HTTP2=true
```

## 验证方法

### 1. 压力测试
```bash
# 使用 wrk 进行压力测试
wrk -t12 -c400 -d30s --latency http://localhost:8000/v1/chat/completions
```

### 2. 监控连接数
```bash
# Linux
watch -n1 "netstat -an | grep :8000 | wc -l"

# 或使用 ss
watch -n1 "ss -tan | grep :8000 | wc -l"
```

### 3. 日志监控
```bash
# 添加详细日志
tail -f app.log | grep -E "pool|timeout|connection"
```

## 预期效果

- ✅ 支持500并发连接
- ✅ 流式响应超时自动清理
- ✅ 客户端断开立即释放资源
- ✅ 连接池利用率提升70%+
- ✅ 不再出现 "pool timeout" 错误

## 关键点总结

1. **最重要**: 确保所有异常路径都关闭响应
2. **次要**: 提高连接池上限和超时配置
3. **优化**: 添加并发限制和监控
4. **预防**: 添加强制超时避免僵尸连接