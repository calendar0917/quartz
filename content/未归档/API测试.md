## 信息收集
API 端点：

```http
GET /api/books HTTP/1.1
Host: example.com
```

识别出端点后，需要确定与这些端点交互的方式。这能让你构造有效的 HTTP 请求来测试 API。例如，你需要查清以下信息：

- API 处理的输入数据，包括必填参数和可选参数；
- API 接受的请求类型，包括支持的 HTTP 方法和媒体格式；
- 速率限制（Rate limits）和认证机制。