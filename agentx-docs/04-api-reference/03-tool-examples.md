# 工具调用示例

本章提供 AgentX 平台工具调用的详细示例，包括内置工具使用、自定义工具集成、错误处理、性能优化等实用场景。

## 内置工具示例

### 1. 天气查询工具

#### 基本使用
```python
import requests

def get_weather_example():
    # 1. 获取访问令牌
    token = get_auth_token()
    
    # 2. 准备请求
    url = "http://localhost:8080/api/v1/agents/process"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}"
    }
    
    # 3. 发送请求
    payload = {
        "agentType": "general",
        "message": "北京天气怎么样？",
        "sessionId": "weather-session-001"
    }
    
    response = requests.post(url, json=payload, headers=headers)
    result = response.json()
    
    if result["success"]:
        # 4. 解析响应
        message = result["data"]["message"]
        tool_calls = result["data"].get("toolCalls", [])
        
        print(f"Agent 回复: {message}")
        
        for tool_call in tool_calls:
            if tool_call["toolName"] == "get_weather":
                weather_data = tool_call["response"]
                print(f"天气数据: {weather_data}")
    else:
        print(f"请求失败: {result['error']['message']}")

def get_auth_token():
    """获取认证令牌"""
    response = requests.post(
        "http://localhost:8080/api/v1/auth/login",
        json={"username": "admin", "password": "admin123"}
    )
    return response.json()["data"]["token"]
```

#### 直接调用工具
```python
def call_weather_directly():
    """直接调用天气工具（绕过 Agent）"""
    token = get_auth_token()
    
    url = "http://localhost:8080/api/v1/tools/get-weather/execute"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}"
    }
    
    payload = {
        "parameters": {
            "city": "上海",
            "unit": "celsius"
        }
    }
    
    response = requests.post(url, json=payload, headers=headers)
    result = response.json()
    
    if result["success"]:
        weather = result["data"]
        print(f"城市: {weather['city']}")
        print(f"温度: {weather['temperature']}°{weather['unit']}")
        print(f"天气: {weather['condition']}")
        print(f"湿度: {weather['humidity']}")
        print(f"风速: {weather['windSpeed']}")
    else:
        print(f"工具调用失败: {result['error']['message']}")
```

#### 批量查询多个城市
```python
def batch_weather_query():
    """批量查询多个城市的天气"""
    cities = ["北京", "上海", "广州", "深圳", "杭州"]
    weather_results = []
    
    for city in cities:
        response = call_weather_tool(city)
        if response["success"]:
            weather_results.append({
                "city": city,
                "data": response["data"]
            })
        else:
            weather_results.append({
                "city": city,
                "error": response["error"]["message"]
            })
    
    # 生成汇总报告
    report = generate_weather_report(weather_results)
    print(report)

def call_weather_tool(city):
    """调用天气工具（封装）"""
    token = get_auth_token()
    
    response = requests.post(
        "http://localhost:8080/api/v1/tools/get-weather/execute",
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Bearer {token}"
        },
        json={"parameters": {"city": city}}
    )
    
    return response.json()
```

### 2. 文档解析工具

#### 解析本地文件
```python
import base64

def parse_local_document():
    """解析本地文档文件"""
    token = get_auth_token()
    
    # 读取文件并 base64 编码
    file_path = "/path/to/document.pdf"
    with open(file_path, "rb") as f:
        file_content = base64.b64encode(f.read()).decode("utf-8")
    
    url = "http://localhost:8080/api/v1/tools/parse-document/execute"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}"
    }
    
    payload = {
        "parameters": {
            "filePath": file_path,
            "fileContent": file_content,  # base64 编码的文件内容
            "extractMetadata": True,
            "maxLength": 5000
        }
    }
    
    response = requests.post(url, json=payload, headers=headers)
    result = response.json()
    
    if result["success"]:
        document_info = result["data"]
        
        print(f"文件名: {document_info['fileName']}")
        print(f"文件大小: {document_info['fileSize']} bytes")
        print(f"文件类型: {document_info['fileType']}")
        print(f"内容长度: {document_info['contentLength']} 字符")
        
        # 显示前 500 字符
        content_preview = document_info["content"][:500]
        print(f"内容预览:\n{content_preview}")
        
        # 显示元数据
        if document_info.get("metadata"):
            print("元数据:")
            for key, value in document_info["metadata"].items():
                print(f"  {key}: {value}")
    else:
        print(f"文档解析失败: {result['error']['message']}")
```

#### 解析远程文档
```python
def parse_remote_document():
    """解析远程 URL 文档"""
    token = get_auth_token()
    
    url = "http://localhost:8080/api/v1/tools/parse-document/execute"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}"
    }
    
    payload = {
        "parameters": {
            "url": "https://example.com/document.pdf",
            "extractMetadata": True,
            "extractText": True,
            "language": "zh-CN"
        }
    }
    
    response = requests.post(url, json=payload, headers=headers)
    result = response.json()
    
    if result["success"]:
        # 处理解析结果
        process_document_result(result["data"])
    else:
        handle_document_error(result["error"])
```

#### 批量文档处理
```python
import concurrent.futures
from typing import List, Dict

def batch_document_processing(documents: List[Dict]) -> List[Dict]:
    """批量处理多个文档"""
    token = get_auth_token()
    results = []
    
    def process_document(doc):
        """处理单个文档"""
        try:
            response = requests.post(
                "http://localhost:8080/api/v1/tools/parse-document/execute",
                headers={
                    "Content-Type": "application/json",
                    "Authorization": f"Bearer {token}"
                },
                json={
                    "parameters": {
                        "filePath": doc["path"],
                        "extractMetadata": doc.get("extractMetadata", True),
                        "maxLength": doc.get("maxLength", 10000)
                    }
                },
                timeout=30
            )
            
            result = response.json()
            return {
                "docId": doc["id"],
                "success": result["success"],
                "data": result.get("data"),
                "error": result.get("error")
            }
        except Exception as e:
            return {
                "docId": doc["id"],
                "success": False,
                "error": {"message": str(e)}
            }
    
    # 使用线程池并行处理
    with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
        future_to_doc = {
            executor.submit(process_document, doc): doc
            for doc in documents
        }
        
        for future in concurrent.futures.as_completed(future_to_doc):
            doc = future_to_doc[future]
            try:
                result = future.result()
                results.append(result)
            except Exception as e:
                results.append({
                    "docId": doc["id"],
                    "success": False,
                    "error": {"message": str(e)}
                })
    
    # 生成处理报告
    generate_processing_report(results)
    return results
```

### 3. 计算工具

#### 基本计算
```python
def calculate_example():
    """计算工具示例"""
    token = get_auth_token()
    
    calculations = [
        {"expression": "2 + 3 * 4"},
        {"expression": "sin(30) + cos(60)"},
        {"expression": "sqrt(16) + log(100)"},
        {"expression": "max(10, 20, 30)"},
        {"expression": "round(3.14159, 2)"}
    ]
    
    for calc in calculations:
        response = requests.post(
            "http://localhost:8080/api/v1/tools/calculate/execute",
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {token}"
            },
            json={"parameters": calc}
        )
        
        result = response.json()
        if result["success"]:
            print(f"{calc['expression']} = {result['data']['result']}")
        else:
            print(f"计算失败 {calc['expression']}: {result['error']['message']}")
```

#### 复杂公式计算
```python
def complex_calculation():
    """复杂公式计算"""
    token = get_auth_token()
    
    # 财务计算示例
    financial_formulas = [
        {
            "expression": "PV(0.05, 10, -1000)",
            "description": "现值计算：年利率5%，10年，每年支付1000"
        },
        {
            "expression": "FV(0.05, 10, -1000)",
            "description": "终值计算"
        },
        {
            "expression": "PMT(0.05/12, 30*12, -500000)",
            "description": "月供计算：贷款50万，年利率5%，30年"
        }
    ]
    
    for formula in financial_formulas:
        response = requests.post(
            "http://localhost:8080/api/v1/tools/finance-calculate/execute",
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {token}"
            },
            json={
                "parameters": {
                    "formula": formula["expression"],
                    "variables": {},
                    "precision": 2
                }
            }
        )
        
        result = response.json()
        if result["success"]:
            calculation = result["data"]
            print(f"{formula['description']}:")
            print(f"  公式: {formula['expression']}")
            print(f"  结果: {calculation['result']}")
            print(f"  单位: {calculation.get('unit', '')}")
            print(f"  说明: {calculation.get('explanation', '')}")
        else:
            print(f"财务计算失败: {result['error']['message']}")
```

### 4. 数据查询工具

#### 数据库查询
```python
def database_query_example():
    """数据库查询示例"""
    token = get_auth_token()
    
    queries = [
        {
            "name": "用户统计",
            "query": "SELECT COUNT(*) as user_count, status FROM users GROUP BY status",
            "database": "user_db",
            "timeout": 30
        },
        {
            "name": "订单分析",
            "query": """
                SELECT 
                    DATE(order_date) as day,
                    COUNT(*) as order_count,
                    SUM(amount) as total_amount
                FROM orders 
                WHERE order_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
                GROUP BY DATE(order_date)
                ORDER BY day DESC
            """,
            "database": "order_db",
            "format": "json"  # json, csv, table
        }
    ]
    
    for query_info in queries:
        response = requests.post(
            "http://localhost:8080/api/v1/tools/database-query/execute",
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {token}"
            },
            json={
                "parameters": query_info,
                "metadata": {
                    "requestId": f"db-query-{query_info['name']}",
                    "userId": "analyst-001"
                }
            }
        )
        
        result = response.json()
        if result["success"]:
            query_result = result["data"]
            print(f"查询: {query_info['name']}")
            print(f"执行时间: {query_result['executionTime']}ms")
            print(f"返回行数: {query_result['rowCount']}")
            
            # 根据格式显示结果
            if query_info.get("format") == "json":
                for row in query_result["rows"]:
                    print(f"  {row}")
            elif query_info.get("format") == "table":
                print(query_result["formattedTable"])
        else:
            print(f"查询失败: {result['error']['message']}")
```

#### API 数据获取
```python
def api_data_fetch():
    """从外部 API 获取数据"""
    token = get_auth_token()
    
    api_requests = [
        {
            "name": "股票行情",
            "endpoint": "https://api.stockdata.com/v1/quote",
            "method": "GET",
            "params": {
                "symbols": "AAPL,GOOGL,MSFT",
                "api_token": "${STOCK_API_KEY}"
            },
            "cache": True,
            "cacheTtl": 300  # 缓存5分钟
        },
        {
            "name": "汇率查询",
            "endpoint": "https://api.exchangerate-api.com/v4/latest/USD",
            "method": "GET",
            "transform": {
                "extract": ["rates.CNY", "rates.EUR", "rates.JPY"],
                "rename": {
                    "rates.CNY": "usd_to_cny",
                    "rates.EUR": "usd_to_eur",
                    "rates.JPY": "usd_to_jpy"
                }
            }
        }
    ]
    
    for request_info in api_requests:
        response = requests.post(
            "http://localhost:8080/api/v1/tools/api-fetch/execute",
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {token}"
            },
            json={
                "parameters": request_info,
                "options": {
                    "timeout": 10000,
                    "retry": {
                        "attempts": 3,
                        "delay": 1000
                    }
                }
            }
        )
        
        result = response.json()
        if result["success"]:
            api_data = result["data"]
            print(f"API: {request_info['name']}")
            print(f"状态码: {api_data['statusCode']}")
            print(f"响应时间: {api_data['responseTime']}ms")
            print(f"数据: {api_data['data']}")
            
            if api_data.get("cached"):
                print("  (来自缓存)")
        else:
            print(f"API 调用失败: {result['error']['message']}")
```

## 自定义工具调用

### 1. 自定义工具注册和调用

#### 注册自定义工具
```python
def register_custom_tool():
    """注册自定义工具到 AgentX"""
    token = get_auth_token()
    
    tool_definition = {
        "name": "custom-data-processor",
        "description": "自定义数据处理工具，用于清洗和转换数据",
        "className": "com.agentx.tools.custom.DataProcessorTool",
        "config": {
            "batchSize": 100,
            "timeout": 30000,
            "retryAttempts": 3
        },
        "parameters": {
            "inputData": "输入数据，JSON格式",
            "operation": "操作类型：clean, transform, validate",
            "rules": "处理规则，JSON格式",
            "outputFormat": "输出格式：json, csv, xml"
        },
        "dependencies": [
            {
                "groupId": "com.example",
                "artifactId": "data-processor",
                "version": "1.2.0"
            }
        ]
    }
    
    response = requests.post(
        "http://localhost:8080/api/v1/tools",
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Bearer {token}"
        },
        json=tool_definition
    )
    
    result = response.json()
    if result["success"]:
        print(f"工具注册成功: {result['data']['name']}")
        print(f"工具ID: {result['data']['id']}")
        print(f"注册时间: {result['data']['registeredAt']}")
    else:
        print(f"工具注册失败: {result['error']['message']}")
```

#### 调用自定义工具
```python
def call_custom_tool():
    """调用自定义数据处理工具"""
    token = get_auth_token()
    
    # 准备数据
    input_data = {
        "users": [
            {"id": 1, "name": "Alice", "email": "alice@example.com", "age": 25},
            {"id": 2, "name": "Bob", "email": "bob@example", "age": "30"},  # 邮箱格式错误
            {"id": 3, "name": "Charlie", "email": "charlie@example.com", "age": 35}
        ]
    }
    
    processing_rules = {
        "validation": {
            "email": {"regex": "^[^@]+@[^@]+\\.[^@]+$"},
            "age": {"type": "integer", "min": 18, "max": 100}
        },
        "cleaning": {
            "trimFields": ["name", "email"],
            "defaults": {"status": "active"}
        },
        "transformation": {
            "addFields": {
                "ageGroup": {
                    "expression": "age < 30 ? 'young' : age < 50 ? 'middle' : 'senior'"
                }
            }
        }
    }
    
    response = requests.post(
        "http://localhost:8080/api/v1/tools/custom-data-processor/execute",
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Bearer {token}"
        },
        json={
            "parameters": {
                "inputData": json.dumps(input_data),
                "operation": "clean",
                "rules": json.dumps(processing_rules),
                "outputFormat": "json"
            },
            "metadata": {
                "requestId": "data-clean-001",
                "userId": "data-engineer-001",
                "pipeline": "user-data-processing"
            }
        }
    )
    
    result = response.json()
    if result["success"]:
        processed_data = result["data"]
        print(f"处理成功:")
        print(f"原始记录数: {processed_data['stats']['originalCount']}")
        print(f"有效记录数: {processed_data['stats']['validCount']}")
        print(f"无效记录数: {processed_data['stats']['invalidCount']}")
        print(f"处理时间: {processed_data['stats']['processingTime']}ms")
        
        if processed_data.get("invalidRecords"):
            print("无效记录:")
            for record in processed_data["invalidRecords"]:
                print(f"  ID {record['id']}: {record['errors']}")
        
        print("处理后的数据:")
        print(json.dumps(processed_data["outputData"], indent=2, ensure_ascii=False))
    else:
        print(f"数据处理失败: {result['error']['message']}")
```

### 2. 工具组合使用

#### 工作流式工具调用
```python
def workflow_tool_chain():
    """工作流式的工具链调用"""
    token = get_auth_token()
    
    # 定义工具链
    tool_chain = [
        {
            "name": "data-fetcher",
            "parameters": {
                "source": "database",
                "query": "SELECT * FROM sales WHERE date >= '2026-01-01'",
                "format": "json"
            },
            "outputMapping": {
                "data": "salesData"
            }
        },
        {
            "name": "data-cleaner",
            "parameters": {
                "inputData": "{{salesData}}",
                "operation": "clean",
                "rules": {
                    "removeNulls": True,
                    "standardizeDates": True,
                    "validateAmounts": True
                }
            },
            "dependsOn": ["data-fetcher"],
            "outputMapping": {
                "cleanedData": "cleanSalesData"
            }
        },
        {
            "name": "data-analyzer",
            "parameters": {
                "data": "{{cleanSalesData}}",
                "analysisType": "summary",
                "metrics": ["total", "average", "max", "min", "count"]
            },
            "dependsOn": ["data-cleaner"],
            "outputMapping": {
                "analysisResult": "salesAnalysis"
            }
        },
        {
            "name": "report-generator",
            "parameters": {
                "data": "{{salesAnalysis}}",
                "template": "sales-report",
                "format": "html"
            },
            "dependsOn": ["data-analyzer"],
            "outputMapping": {
                "report": "finalReport"
            }
        }
    ]
    
    # 执行工具链
    execution_id = start_tool_chain_execution(tool_chain, token)
    
    # 监控执行状态
    monitor_execution(execution_id, token)
    
    # 获取最终结果
    final_result = get_execution_result(execution_id, token)
    
    if final_result["success"]:
        print("工具链执行成功!")
        print(f"执行ID: {execution_id}")
        print(f"总耗时: {final_result['data']['totalDuration']}ms")
        print(f"成功步骤: {final_result['data']['successfulSteps']}")
        print(f"失败步骤: {final_result['data']['failedSteps']}")
        
        # 输出最终报告
        report = final_result["data"]["outputs"]["finalReport"]
        save_report(report, "sales-report.html")
    else:
        print(f"工具链执行失败: {final_result['error']['message']}")
```

#### 条件工具调用
```python
def conditional_tool_execution():
    """根据条件选择不同的工具"""
    token = get_auth_token()
    
    def analyze_text(text):
        """分析文本内容并选择合适的工具"""
        
        # 1. 先判断文本类型
        response = requests.post(
            "http://localhost:8080/api/v1/tools/text-analyzer/execute",
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {token}"
            },
            json={
                "parameters": {
                    "text": text,
                    "analysisTypes": ["category", "sentiment", "language"]
                }
            }
        )
        
        analysis_result = response.json()
        if not analysis_result["success"]:
            return {"error": "文本分析失败"}
        
        analysis_data = analysis_result["data"]
        
        # 2. 根据分析结果选择工具
        category = analysis_data.get("category")
        sentiment = analysis_data.get("sentiment", {}).get("score", 0)
        
        if category == "weather_query":
            # 提取城市信息并调用天气工具
            city = extract_city_from_text(text)
            if city:
                return call_weather_tool(city)
            else:
                return {"error": "无法提取城市信息"}
        
        elif category == "calculation":
            # 提取数学表达式并调用计算工具
            expression = extract_expression_from_text(text)
            if expression:
                return call_calculation_tool(expression)
            else:
                return {"error": "无法提取数学表达式"}
        
        elif sentiment < -0.5:
            # 负面情感，调用情感支持工具
            return call_sentiment_support_tool(text)
        
        elif "翻译" in text or "translate" in text.lower():
            # 翻译请求
            return call_translation_tool(text)
        
        else:
            # 默认使用通用 Agent
            return call_general_agent(text)
    
    # 测试不同文本
    test_texts = [
        "北京天气怎么样？",
        "计算一下 2 加 3 乘 4 等于多少",
        "我今天心情很不好",
        "请把这句话翻译成英文",
        "告诉我一些关于人工智能的信息"
    ]
    
    for text in test_texts:
        print(f"\n处理文本: {text}")
        result = analyze_text(text)
        
        if "error" in result:
            print(f"  错误: {result['error']}")
        else:
            print(f"  结果: {result.get('message', result)}")
```

## 错误处理示例

### 1. 基本错误处理

#### 处理工具调用错误
```python
def safe_tool_call(tool_name, parameters):
    """安全的工具调用，包含错误处理"""
    token = get_auth_token()
    
    try:
        response = requests.post(
            f"http://localhost:8080/api/v1/tools/{tool_name}/execute",
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {token}"
            },
            json={"parameters": parameters},
            timeout=30
        )
        
        # 检查 HTTP 状态码
        if response.status_code != 200:
            return {
                "success": False,
                "error": {
                    "type": "http_error",
                    "code": response.status_code,
                    "message": f"HTTP error: {response.status_code}"
                }
            }
        
        result = response.json()
        
        # 检查 API 响应是否成功
        if not result.get("success"):
            error_info = result.get("error", {})
            return {
                "success": False,
                "error": {
                    "type": "api_error",
                    "code": error_info.get("code"),
                    "message": error_info.get("message"),
                    "details": error_info.get("details")
                }
            }
        
        return {
            "success": True,
            "data": result.get("data"),
            "metadata": {
                "latency": result.get("latency", 0),
                "requestId": result.get("requestId")
            }
        }
        
    except requests.exceptions.Timeout:
        return {
            "success": False,
            "error": {
                "type": "timeout",
                "message": "请求超时"
            }
        }
        
    except requests.exceptions.ConnectionError:
        return {
            "success": False,
            "error": {
                "type": "connection_error",
                "message": "连接失败，请检查网络"
            }
        }
        
    except requests.exceptions.RequestException as e:
        return {
            "success": False,
            "error": {
                "type": "request_error",
                "message": str(e)
            }
        }
        
    except json.JSONDecodeError:
        return {
            "success": False,
            "error": {
                "type": "json_error",
                "message": "响应解析失败"
            }
        }
        
    except Exception as e:
        return {
            "success": False,
            "error": {
                "type": "unknown_error",
                "message": f"未知错误: {str(e)}"
            }
        }
```

#### 重试机制
```python
import time
from typing import Optional, Dict, Any

def retry_tool_call(tool_name: str, 
                   parameters: Dict[str, Any],
                   max_attempts: int = 3,
                   base_delay: float = 1.0) -> Optional[Dict[str, Any]]:
    """带重试机制的工具调用"""
    
    attempt = 0
    last_error = None
    
    while attempt < max_attempts:
        attempt += 1
        
        print(f"第 {attempt} 次尝试调用工具 {tool_name}...")
        
        result = safe_tool_call(tool_name, parameters)
        
        if result["success"]:
            print(f"工具调用成功 (第 {attempt} 次尝试)")
            return result
        
        # 检查错误类型，决定是否重试
        error_type = result["error"]["type"]
        last_error = result["error"]
        
        # 某些错误不应该重试
        if error_type in ["validation_error", "permission_error"]:
            print(f"不重试的错误类型: {error_type}")
            break
        
        # 如果是最后一次尝试，不再等待
        if attempt >= max_attempts:
            print(f"已达到最大重试次数 ({max_attempts})")
            break
        
        # 指数退避
        delay = base_delay * (2 ** (attempt - 1))
        print(f"等待 {delay:.1f} 秒后重试...")
        time.sleep(delay)
    
    # 所有尝试都失败
    return {
        "success": False,
        "error": last_error,
        "attempts": attempt
    }
```

### 2. 错误分类和处理策略

#### 错误分类器
```python
def classify_and_handle_error(error_info: Dict[str, Any]) -> Dict[str, Any]:
    """分类错误并采取相应的处理策略"""
    
    error_type = error_info.get("type", "unknown")
    error_code = error_info.get("code")
    error_message = error_info.get("message", "")
    
    # 根据错误类型分类
    if error_type == "validation_error":
        return handle_validation_error(error_info)
    
    elif error_type == "authentication_error":
        return handle_authentication_error(error_info)
    
    elif error_type == "rate_limit_error":
        return handle_rate_limit_error(error_info)
    
    elif error_type == "timeout":
        return handle_timeout_error(error_info)
    
    elif error_type == "connection_error":
        return handle_connection_error(error_info)
    
    elif error_type == "resource_not_found":
        return handle_resource_not_found_error(error_info)
    
    elif error_type == "server_error":
        return handle_server_error(error_info)
    
    else:
        return handle_unknown_error(error_info)

def handle_validation_error(error_info):
    """处理验证错误"""
    details = error_info.get("details", {})
    field_errors = details.get("fieldErrors", [])
    
    suggestions = []
    for field_error in field_errors:
        field = field_error.get("field")
        message = field_error.get("message")
        suggestions.append(f"字段 '{field}': {message}")
    
    return {
        "handled": True,
        "action": "retry_with_correction",
        "message": "参数验证失败，请修正以下问题:",
        "suggestions": suggestions,
        "can_retry": True,
        "requires_correction": True
    }

def handle_rate_limit_error(error_info):
    """处理限流错误"""
    retry_after = error_info.get("details", {}).get("retryAfter", 60)
    
    return {
        "handled": True,
        "action": "wait_and_retry",
        "message": f"请求过于频繁，请等待 {retry_after} 秒后重试",
        "retry_after": retry_after,
        "can_retry": True,
        "requires_wait": True
    }

def handle_timeout_error(error_info):
    """处理超时错误"""
    return {
        "handled": True,
        "action": "retry_with_timeout",
        "message": "请求超时，建议增加超时时间或简化请求",
        "can_retry": True,
        "suggested_timeout": 60,  # 建议增加到60秒
        "max_retries": 2
    }
```

### 3. 错误恢复策略

#### 降级策略
```python
def call_tool_with_fallback(tool_name, parameters, fallback_tools=None):
    """带降级策略的工具调用"""
    
    # 主要工具调用
    result = safe_tool_call(tool_name, parameters)
    
    if result["success"]:
        return result
    
    # 主要工具失败，尝试降级
    if fallback_tools:
        for fallback_tool in fallback_tools:
            print(f"主要工具失败，尝试降级到: {fallback_tool}")
            
            # 可能需要调整参数以适应降级工具
            adjusted_params = adjust_parameters_for_fallback(
                parameters, fallback_tool
            )
            
            fallback_result = safe_tool_call(fallback_tool, adjusted_params)
            
            if fallback_result["success"]:
                print(f"降级工具 {fallback_tool} 调用成功")
                return {
                    **fallback_result,
                    "metadata": {
                        **fallback_result.get("metadata", {}),
                        "primary_tool_failed": True,
                        "fallback_tool_used": fallback_tool
                    }
                }
    
    # 所有降级方案都失败
    print("所有工具调用均失败")
    return result  # 返回原始错误
```

#### 断路器模式
```python
class CircuitBreaker:
    """断路器模式实现"""
    
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def execute(self, tool_call_func, *args, **kwargs):
        """执行工具调用，受断路器保护"""
        
        if self.state == "OPEN":
            # 检查是否应该尝试恢复
            if self.should_attempt_recovery():
                self.state = "HALF_OPEN"
                print("断路器进入半开状态，尝试恢复")
            else:
                raise Exception("断路器已打开，请求被阻断")
        
        try:
            # 执行实际调用
            result = tool_call_func(*args, **kwargs)
            
            if self.state == "HALF_OPEN":
                # 半开状态下成功，重置断路器
                self.reset()
            
            return result
            
        except Exception as e:
            # 调用失败
            self.record_failure()
            raise e
    
    def record_failure(self):
        """记录失败"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
            print(f"断路器打开，失败次数: {self.failure_count}")
    
    def should_attempt_recovery(self):
        """检查是否应该尝试恢复"""
        if self.state != "OPEN" or not self.last_failure_time:
            return False
        
        elapsed = time.time() - self.last_failure_time
        return elapsed >= self.recovery_timeout
    
    def reset(self):
        """重置断路器"""
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"
        print("断路器已重置")

# 使用示例
breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=60)

try:
    result = breaker.execute(
        lambda: safe_tool_call("external-api", {"param": "value"})
    )
    print(f"调用成功: {result}")
except Exception as e:
    print(f"调用失败: {e}")
```

## 性能优化示例

### 1. 批量处理优化

#### 批量工具调用
```python
def batch_tool_calls(tool_name, items, batch_size=10):
    """批量调用工具，提高效率"""
    
    all_results = []
    total_items = len(items)
    
    for i in range(0, total_items, batch_size):
        batch = items[i:i + batch_size]
        batch_number = i // batch_size + 1
        
        print(f"处理批次 {batch_number}，共 {len(batch)} 项...")
        
        # 准备批量参数
        batch_params = {
            "items": batch,
            "batchId": f"batch-{batch_number}",
            "options": {
                "parallel": True,
                "timeout": len(batch) * 2000  # 根据批次大小调整超时
            }
        }
        
        # 调用批量版本的工具
        result = safe_tool_call(f"{tool_name}-batch", batch_params)
        
        if result["success"]:
            batch_results = result["data"]["results"]
            all_results.extend(batch_results)
            
            batch_stats = result["data"]["stats"]
            print(f"  批次 {batch_number} 完成: "
                  f"成功 {batch_stats['successful']}，"
                  f"失败 {batch_stats['failed']}，"
                  f"耗时 {batch_stats['processingTime']}ms")
        else:
            print(f"  批次 {batch_number} 失败: {result['error']['message']}")
            # 可以记录失败，继续处理下一批
    
    # 生成总体统计
    successful = sum(1 for r in all_results if r.get("success"))
    failed = total_items - successful
    
    return {
        "success": True,
        "data": {
            "results": all_results,
            "stats": {
                "totalItems": total_items,
                "successful": successful,
                "failed": failed,
                "successRate": successful / total_items if total_items > 0 else 0
            }
        }
    }
```

#### 并行处理
```python
import asyncio
import aiohttp
from typing import List, Dict, Any

async def parallel_tool_calls(tool_name: str, 
                             requests: List[Dict[str, Any]],
                             max_concurrent: int = 5) -> List[Dict[str, Any]]:
    """并行调用工具"""
    
    async def call_tool(session, request):
        """单个工具调用"""
        try:
            async with session.post(
                f"http://localhost:8080/api/v1/tools/{tool_name}/execute",
                headers={
                    "Content-Type": "application/json",
                    "Authorization": f"Bearer {get_auth_token()}"
                },
                json={"parameters": request["parameters"]},
                timeout=aiohttp.ClientTimeout(total=30)
            ) as response:
                
                result = await response.json()
                return {
                    "requestId": request.get("id"),
                    "success": result.get("success", False),
                    "data": result.get("data"),
                    "error": result.get("error"),
                    "latency": result.get("latency", 0)
                }
                
        except Exception as e:
            return {
                "requestId": request.get("id"),
                "success": False,
                "error": {"message": str(e)}
            }
    
    # 使用信号量控制并发数
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def semaphore_call(session, request):
        """带信号量控制的调用"""
        async with semaphore:
            return await call_tool(session, request)
    
    # 创建会话并并行执行
    async with aiohttp.ClientSession() as session:
        tasks = [semaphore_call(session, req) for req in requests]
        results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # 处理结果
    final_results = []
    for result in results:
        if isinstance(result, Exception):
            final_results.append({
                "success": False,
                "error": {"message": str(result)}
            })
        else:
            final_results.append(result)
    
    return final_results

# 使用示例
async def example_parallel_calls():
    requests = [
        {"id": "req-1", "parameters": {"city": "北京"}},
        {"id": "req-2", "parameters": {"city": "上海"}},
        {"id": "req-3", "parameters": {"city": "广州"}},
        {"id": "req-4", "parameters": {"city": "深圳"}},
        {"id": "req-5", "parameters": {"city": "杭州"}},
    ]
    
    results = await parallel_tool_calls("get-weather", requests, max_concurrent=3)
    
    for result in results:
        if result["success"]:
            print(f"请求 {result['requestId']} 成功: {result['data']['temperature']}°C")
        else:
            print(f"请求 {result['requestId']} 失败: {result['error']['message']}")
```

### 2. 缓存优化

#### 客户端缓存
```python
import functools
from typing import Any, Callable
from datetime import datetime, timedelta

class ToolCache:
    """工具调用缓存"""
    
    def __init__(self, default_ttl=300):  # 默认5分钟
        self.cache = {}
        self.default_ttl = default_ttl
    
    def get(self, key: str) -> Any:
        """获取缓存值"""
        entry = self.cache.get(key)
        if not entry:
            return None
        
        # 检查是否过期
        if datetime.now() > entry["expires_at"]:
            del self.cache[key]
            return None
        
        return entry["value"]
    
    def set(self, key: str, value: Any, ttl: int = None):
        """设置缓存值"""
        ttl = ttl or self.default_ttl
        self.cache[key] = {
            "value": value,
            "expires_at": datetime.now() + timedelta(seconds=ttl),
            "cached_at": datetime.now()
        }
    
    def cached_call(self, ttl: int = None):
        """缓存装饰器"""
        def decorator(func: Callable):
            @functools.wraps(func)
            def wrapper(*args, **kwargs):
                # 生成缓存键
                cache_key = self._generate_key(func, args, kwargs)
                
                # 尝试从缓存获取
                cached_result = self.get(cache_key)
                if cached_result is not None:
                    return {
                        **cached_result,
                        "metadata": {
                            **cached_result.get("metadata", {}),
                            "cached": True,
                            "cache_hit": True
                        }
                    }
                
                # 调用实际函数
                result = func(*args, **kwargs)
                
                # 如果成功，缓存结果
                if result.get("success"):
                    self.set(cache_key, result, ttl)
                    result["metadata"] = {
                        **result.get("metadata", {}),
                        "cached": False,
                        "cache_hit": False
                    }
                
                return result
            
            return wrapper
        return decorator
    
    def _generate_key(self, func, args, kwargs):
        """生成缓存键"""
        import hashlib
        import json
        
        key_data = {
            "func": func.__name__,
            "args": args,
            "kwargs": kwargs
        }
        
        key_str = json.dumps(key_data, sort_keys=True, default=str)
        return hashlib.md5(key_str.encode()).hexdigest()

# 使用示例
cache = ToolCache(default_ttl=300)  # 5分钟缓存

@cache.cached_call(ttl=600)  # 10分钟缓存
def get_city_weather(city: str):
    """获取城市天气（带缓存）"""
    return safe_tool_call("get-weather", {"city": city})

# 第一次调用（未缓存）
result1 = get_city_weather("北京")
print(f"第一次调用: 缓存命中={result1['metadata']['cache_hit']}")

# 第二次调用（缓存命中）
result2 = get_city_weather("北京")
print(f"第二次调用: 缓存命中={result2['metadata']['cache_hit']}")

# 不同城市（缓存未命中）
result3 = get_city_weather("上海")
print(f"第三次调用: 缓存命中={result3['metadata']['cache_hit']}")
```

### 3. 预加载和预热

#### 工具预热
```python
def warmup_tools(tool_names, concurrency=3):
    """预热工具，减少首次调用延迟"""
    
    def warmup_tool(tool_name):
        """预热单个工具"""
        try:
            # 执行一个简单的调用
            result = safe_tool_call(tool_name, get_warmup_parameters(tool_name))
            
            if result["success"]:
                return {
                    "tool": tool_name,
                    "status": "warmed_up",
                    "latency": result.get("metadata", {}).get("latency", 0)
                }
            else:
                return {
                    "tool": tool_name,
                    "status": "failed",
                    "error": result["error"]["message"]
                }
        except Exception as e:
            return {
                "tool": tool_name,
                "status": "error",
                "error": str(e)
            }
    
    def get_warmup_parameters(tool_name):
        """获取预热参数"""
        warmup_params = {
            "get-weather": {"city": "北京"},
            "calculate": {"expression": "1 + 1"},
            "parse-document": {"test": True},  # 测试模式
            "database-query": {"testQuery": True}
        }
        return warmup_params.get(tool_name, {})
    
    # 并行预热
    with concurrent.futures.ThreadPoolExecutor(max_workers=concurrency) as executor:
        future_to_tool = {
            executor.submit(warmup_tool, tool_name): tool_name
            for tool_name in tool_names
        }
        
        results = []
        for future in concurrent.futures.as_completed(future_to_tool):
            tool_name = future_to_tool[future]
            try:
                result = future.result()
                results.append(result)
                print(f"工具预热: {tool_name} - {result['status']}")
            except Exception as e:
                results.append({
                    "tool": tool_name,
                    "status": "exception",
                    "error": str(e)
                })
                print(f"工具预热异常: {tool_name} - {e}")
    
    # 统计
    warmed_up = sum(1 for r in results if r["status"] == "warmed_up")
    failed = sum(1 for r in results if r["status"] in ["failed", "error"])
    
    print(f"预热完成: {warmed_up} 成功, {failed} 失败")
    return results
```

## 监控和日志示例

### 1. 性能监控

#### 工具调用监控
```python
import time
from dataclasses import dataclass
from typing import Optional, Dict, Any
from contextlib import contextmanager

@dataclass
class ToolCallMetrics:
    """工具调用指标"""
    tool_name: str
    start_time: float
    end_time: Optional[float] = None
    success: Optional[bool] = None
    error_type: Optional[str] = None
    result_size: Optional[int] = None
    
    @property
    def duration(self) -> float:
        if self.end_time:
            return (self.end_time - self.start_time) * 1000  # 毫秒
        return 0
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            "tool_name": self.tool_name,
            "duration_ms": self.duration,
            "success": self.success,
            "error_type": self.error_type,
            "result_size": self.result_size,
            "timestamp": self.start_time
        }

class ToolMonitor:
    """工具调用监控器"""
    
    def __init__(self):
        self.metrics_history = []
        self.call_count = 0
        self.error_count = 0
        self.total_duration = 0
    
    @contextmanager
    def monitor_call(self, tool_name: str):
        """监控上下文管理器"""
        metrics = ToolCallMetrics(tool_name=tool_name, start_time=time.time())
        
        try:
            yield metrics
            metrics.success = True
        except Exception as e:
            metrics.success = False
            metrics.error_type = type(e).__name__
            self.error_count += 1
            raise
        finally:
            metrics.end_time = time.time()
            self._record_metrics(metrics)
    
    def _record_metrics(self, metrics: ToolCallMetrics):
        """记录指标"""
        self.metrics_history.append(metrics)
        self.call_count += 1
        self.total_duration += metrics.duration
        
        # 保持历史记录大小
        if len(self.metrics_history) > 1000:
            self.metrics_history = self.metrics_history[-1000:]
    
    def get_summary(self) -> Dict[str, Any]:
        """获取监控摘要"""
        if not self.metrics_history:
            return {}
        
        recent_metrics = self.metrics_history[-100:]  # 最近100次调用
        
        success_count = sum(1 for m in recent_metrics if m.success)
        error_count = sum(1 for m in recent_metrics if not m.success)
        
        durations = [m.duration for m in recent_metrics if m.end_time]
        
        return {
            "total_calls": self.call_count,
            "recent_calls": len(recent_metrics),
            "success_rate": success_count / len(recent_metrics) if recent_metrics else 0,
            "error_rate": error_count / len(recent_metrics) if recent_metrics else 0,
            "avg_duration_ms": sum(durations) / len(durations) if durations else 0,
            "p95_duration_ms": sorted(durations)[int(len(durations) * 0.95)] if durations else 0,
            "p99_duration_ms": sorted(durations)[int(len(durations) * 0.99)] if durations else 0,
            "total_duration_ms": self.total_duration
        }
    
    def get_tool_stats(self, tool_name: str) -> Dict[str, Any]:
        """获取特定工具的统计"""
        tool_metrics = [m for m in self.metrics_history if m.tool_name == tool_name]
        
        if not tool_metrics:
            return {}
        
        success_count = sum(1 for m in tool_metrics if m.success)
        durations = [m.duration for m in tool_metrics if m.end_time]
        
        return {
            "tool_name": tool_name,
            "call_count": len(tool_metrics),
            "success_count": success_count,
            "error_count": len(tool_metrics) - success_count,
            "success_rate": success_count / len(tool_metrics),
            "avg_duration_ms": sum(durations) / len(durations) if durations else 0,
            "min_duration_ms": min(durations) if durations else 0,
            "max_duration_ms": max(durations) if durations else 0
        }

# 使用示例
monitor = ToolMonitor()

def monitored_tool_call(tool_name, parameters):
    """带监控的工具调用"""
    with monitor.monitor_call(tool_name) as metrics:
        result = safe_tool_call(tool_name, parameters)
        metrics.success = result["success"]
        metrics.result_size = len(str(result.get("data", "")))
        return result

# 定期打印监控摘要
def print_monitoring_summary():
    summary = monitor.get_summary()
    print("\n=== 工具调用监控摘要 ===")
    print(f"总调用次数: {summary['total_calls']}")
    print(f"成功率: {summary['success_rate']:.2%}")
    print(f"平均耗时: {summary['avg_duration_ms']:.2f}ms")
    print(f"P95耗时: {summary['p95_duration_ms']:.2f}ms")
    print(f"P99耗时: {summary['p99_duration_ms']:.2f}ms")
    
    # 打印各工具统计
    tool_names = set(m.tool_name for m in monitor.metrics_history)
    for tool_name in tool_names:
        stats = monitor.get_tool_stats(tool_name)
        if stats:
            print(f"\n{tool_name}:")
            print(f"  调用次数: {stats['call_count']}")
            print(f"  成功率: {stats['success_rate']:.2%}")
            print(f"  平均耗时: {stats['avg_duration_ms']:.2f}ms")
```

### 2. 结构化日志

#### 工具调用日志
```python
import logging
import json
from pythonjsonlogger import jsonlogger

class ToolCallLogger:
    """工具调用结构化日志"""
    
    def __init__(self, name="tool_calls"):
        self.logger = logging.getLogger(name)
        
        # 配置 JSON 格式日志
        log_handler = logging.StreamHandler()
        formatter = jsonlogger.JsonFormatter(
            '%(asctime)s %(name)s %(levelname)s %(message)s'
        )
        log_handler.setFormatter(formatter)
        self.logger.addHandler(log_handler)
        self.logger.setLevel(logging.INFO)
    
    def log_tool_call(self, tool_name: str, parameters: Dict[str, Any], 
                     result: Dict[str, Any], duration_ms: float):
        """记录工具调用日志"""
        
        # 准备日志数据
        log_data = {
            "event": "tool_call",
            "tool_name": tool_name,
            "parameters": self._sanitize_parameters(parameters),
            "success": result.get("success", False),
            "duration_ms": duration_ms,
            "timestamp": time.time()
        }
        
        # 添加结果信息（去除敏感数据）
        if result.get("success"):
            log_data["result_summary"] = self._summarize_result(result.get("data"))
        else:
            error_info = result.get("error", {})
            log_data["error"] = {
                "type": error_info.get("type"),
                "code": error_info.get("code"),
                "message": error_info.get("message")
            }
        
        # 记录日志
        if result.get("success"):
            self.logger.info("Tool call completed", extra=log_data)
        else:
            self.logger.warning("Tool call failed", extra=log_data)
    
    def _sanitize_parameters(self, parameters: Dict[str, Any]) -> Dict[str, Any]:
        """清理参数，移除敏感信息"""
        sanitized = parameters.copy()
        
        # 移除可能的敏感字段
        sensitive_fields = ["password", "token", "secret", "key", "authorization"]
        for field in sensitive_fields:
            if field in sanitized:
                sanitized[field] = "[REDACTED]"
        
        # 限制大字段大小
        for key, value in sanitized.items():
            if isinstance(value, str) and len(value) > 100:
                sanitized[key] = value[:100] + "..."
        
        return sanitized
    
    def _summarize_result(self, result_data: Any) -> Dict[str, Any]:
        """总结结果数据"""
        if not result_data:
            return {}
        
        if isinstance(result_data, dict):
            summary = {}
            for key, value in result_data.items():
                if isinstance(value, (str, int, float, bool)):
                    summary[key] = value
                elif isinstance(value, dict):
                    summary[key] = {"type": "object", "keys": list(value.keys())[:5]}
                elif isinstance(value, list):
                    summary[key] = {"type": "list", "count": len(value), "sample": value[:3]}
                else:
                    summary[key] = {"type": type(value).__name__}
            
            # 限制大小
            if len(str(summary)) > 500:
                return {"summary": "result too large to log"}
            
            return summary
        else:
            return {"type": type(result_data).__name__, "value": str(result_data)[:100]}

# 使用示例
logger = ToolCallLogger()

def logged_tool_call(tool_name, parameters):
    """带日志记录的工具调用"""
    start_time = time.time()
    
    try:
        result = safe_tool_call(tool_name, parameters)
        duration_ms = (time.time() - start_time) * 1000
        
        # 记录日志
        logger.log_tool_call(tool_name, parameters, result, duration_ms)
        
        return result
    except Exception as e:
        duration_ms = (time.time() - start_time) * 1000
        error_result = {
            "success": False,
            "error": {"type": type(e).__name__, "message": str(e)}
        }
        
        logger.log_tool_call(tool_name, parameters, error_result, duration_ms)
        raise
```

## 最佳实践总结

### 1. 工具调用模式
- **批量处理**：尽可能使用批量接口，减少网络开销
- **并行执行**：独立任务使用并行处理提高效率
- **缓存策略**：合理使用缓存，减少重复计算
- **预热机制**：冷启动时预热工具，减少首次延迟

### 2. 错误处理策略
- **分类处理**：根据错误类型采取不同策略
- **重试机制**：临时性错误使用重试，配合指数退避
- **降级方案**：主要工具失败时提供降级方案
- **断路器模式**：防止级联故障，保护系统

### 3. 性能优化
- **监控指标**：监控成功率、延迟、吞吐量等关键指标
- **资源管理**：合理管理连接池、线程池等资源
- **负载均衡**：分布式部署时使用负载均衡
- **容量规划**：基于监控数据进行容量规划

### 4. 安全和可观测性
- **结构化日志**：使用结构化日志便于分析和监控
- **审计跟踪**：记录所有工具调用用于审计
- **敏感数据处理**：避免在日志中记录敏感信息
- **权限控制**：实施最小权限原则

### 5. 代码质量
- **错误处理**：所有工具调用都要有错误处理
- **超时设置**：合理设置超时时间，避免阻塞
- **资源清理**：确保连接、文件等资源正确释放
- **测试覆盖**：为工具调用编写单元测试和集成测试

## 下一步

根据你的需求选择：
- **快速上手**：从内置工具示例开始，体验工具调用
- **深度集成**：参考自定义工具示例，集成业务逻辑
- **生产部署**：查看性能优化和监控示例，确保系统稳定
- **故障排除**：参考错误处理示例，处理常见问题

如需更多帮助，请查看：
- [工具开发指南](../03-development-guide/03-adding-tools.md)
- [API 参考](01-rest-api.md)
- [MCP 协议接口](02-mcp-api.md)
- [社区示例](https://github.com/agentx/examples)