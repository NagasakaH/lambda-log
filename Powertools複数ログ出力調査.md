# Powertools で複数ロググループに出力する方法

## 重要な前提

**Powertools 標準機能では複数ロググループへの出力はサポートされていません。**

Lambda は 1つの関数につき 1つのロググループにのみ出力する仕組みです。実装検討.md のパターン3（階層分離）は**カスタム実装が必要**な構成です。

---

## 実装方法

### 方法1: CloudWatch Logs API を直接呼び出す（推奨）

```csharp
using Amazon.CloudWatchLogs;
using Amazon.CloudWatchLogs.Model;
using AWS.Lambda.Powertools.Logging;
using System.Text.Json;

public class MultiLogGroupLogger
{
    private readonly IAmazonCloudWatchLogs _cwLogs;
    private readonly string _errorLogGroup;
    private readonly string _warnLogGroup;
    
    public MultiLogGroupLogger()
    {
        _cwLogs = new AmazonCloudWatchLogsClient();
        _errorLogGroup = Environment.GetEnvironmentVariable("ERROR_LOG_GROUP") ?? "/app/lambda/errors";
        _warnLogGroup = Environment.GetEnvironmentVariable("WARN_LOG_GROUP") ?? "/app/lambda/warnings";
    }
    
    public async Task LogError(string message, Exception? ex, ILambdaContext context)
    {
        // 通常のPowertools Logger にも出力（デフォルトロググループ）
        Logger.LogError(ex, message);
        
        // ERROR専用ロググループにも出力
        await WriteToLogGroup(_errorLogGroup, "Error", message, context);
    }
    
    public async Task LogWarning(string message, ILambdaContext context)
    {
        Logger.LogWarning(message);
        await WriteToLogGroup(_warnLogGroup, "Warning", message, context);
    }
    
    private async Task WriteToLogGroup(string logGroup, string level, string message, ILambdaContext context)
    {
        var logEntry = new
        {
            timestamp = DateTime.UtcNow.ToString("O"),
            level,
            function_name = context.FunctionName,
            request_id = context.AwsRequestId,
            message
        };
        
        var streamName = $"{context.FunctionName}/{DateTime.UtcNow:yyyy/MM/dd}";
        
        try
        {
            await _cwLogs.PutLogEventsAsync(new PutLogEventsRequest
            {
                LogGroupName = logGroup,
                LogStreamName = streamName,
                LogEvents = new List<InputLogEvent>
                {
                    new() { Timestamp = DateTime.UtcNow, Message = JsonSerializer.Serialize(logEntry) }
                }
            });
        }
        catch (ResourceNotFoundException)
        {
            // ログストリームを作成してリトライ
            await _cwLogs.CreateLogStreamAsync(new CreateLogStreamRequest
            {
                LogGroupName = logGroup,
                LogStreamName = streamName
            });
            await WriteToLogGroup(logGroup, level, message, context);
        }
    }
}
```

### 方法2: ILogger + Custom Provider

.NET の `ILogger` インターフェースを使い、カスタムプロバイダーでログレベル別にルーティング。

```csharp
using Microsoft.Extensions.Logging;

public class MultiDestinationLoggerProvider : ILoggerProvider
{
    private readonly IAmazonCloudWatchLogs _cwLogs;
    private readonly string _errorLogGroup;
    private readonly string _warnLogGroup;

    public MultiDestinationLoggerProvider(string errorLogGroup, string warnLogGroup)
    {
        _cwLogs = new AmazonCloudWatchLogsClient();
        _errorLogGroup = errorLogGroup;
        _warnLogGroup = warnLogGroup;
    }

    public ILogger CreateLogger(string categoryName)
    {
        return new MultiDestinationLogger(_cwLogs, _errorLogGroup, _warnLogGroup, categoryName);
    }

    public void Dispose() { }
}

public class MultiDestinationLogger : ILogger
{
    private readonly IAmazonCloudWatchLogs _cwLogs;
    private readonly string _errorLogGroup;
    private readonly string _warnLogGroup;
    private readonly string _categoryName;

    public MultiDestinationLogger(
        IAmazonCloudWatchLogs cwLogs,
        string errorLogGroup,
        string warnLogGroup,
        string categoryName)
    {
        _cwLogs = cwLogs;
        _errorLogGroup = errorLogGroup;
        _warnLogGroup = warnLogGroup;
        _categoryName = categoryName;
    }

    public IDisposable? BeginScope<TState>(TState state) where TState : notnull => null;

    public bool IsEnabled(LogLevel logLevel) => logLevel >= LogLevel.Warning;

    public void Log<TState>(
        LogLevel logLevel,
        EventId eventId,
        TState state,
        Exception? exception,
        Func<TState, Exception?, string> formatter)
    {
        if (!IsEnabled(logLevel)) return;

        var message = formatter(state, exception);
        var logGroup = logLevel >= LogLevel.Error ? _errorLogGroup : _warnLogGroup;

        // 非同期で書き込み（Fire and forget）
        Task.Run(() => WriteToLogGroupAsync(logGroup, logLevel.ToString(), message));
    }

    private async Task WriteToLogGroupAsync(string logGroup, string level, string message)
    {
        // PutLogEvents の実装（方法1と同様）
    }
}
```

### 方法3: Lambda関数を分離

ログレベル別にLambda関数の設定でロググループを分ける（Terraform例）。

```hcl
# ERROR/CRIT用Lambda
resource "aws_lambda_function" "error_handler" {
  function_name = "my-app-error-handler"
  
  logging_config {
    log_format = "JSON"
    log_group  = aws_cloudwatch_log_group.errors.name
  }
}

# INFO用Lambda
resource "aws_lambda_function" "main" {
  function_name = "my-app-main"
  
  logging_config {
    log_format = "JSON"
    log_group  = aws_cloudwatch_log_group.info.name
  }
}
```

---

## 必要な IAM 権限

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:*:*:log-group:/app/lambda/errors:*",
        "arn:aws:logs:*:*:log-group:/app/lambda/warnings:*"
      ]
    }
  ]
}
```

---

## 参考ドキュメント

### Powertools for AWS Lambda (.NET)

| トピック | URL |
|---------|-----|
| **Logging V2 ドキュメント** | https://docs.aws.amazon.com/powertools/dotnet/core/logging-v2/ |
| **Logger クラス API** | https://docs.aws.amazon.com/powertools/dotnet/api/api/AWS.Lambda.Powertools.Logging.Logger.html |
| **ILogFormatter インターフェース** | https://docs.aws.amazon.com/powertools/dotnet/api/api/AWS.Lambda.Powertools.Logging.ILogFormatter.html |
| **PowertoolsLoggingBuilderExtensions** | https://docs.aws.amazon.com/powertools/dotnet/api/api/AWS.Lambda.Powertools.Logging.PowertoolsLoggingBuilderExtensions.html |
| **Logging 名前空間** | https://docs.aws.amazon.com/powertools/dotnet/api/api/AWS.Lambda.Powertools.Logging.html |
| **LoggingAttribute クラス** | https://docs.aws.amazon.com/powertools/dotnet/api/api/AWS.Lambda.Powertools.Logging.LoggingAttribute.html |
| **公式ドキュメント（トップ）** | https://docs.powertools.aws.dev/lambda/dotnet/ |

### Lambda ロググループ設定

| トピック | URL |
|---------|-----|
| **カスタムロググループ設定** | https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs-loggroups.html |
| **Lambda ログの操作** | https://docs.aws.amazon.com/lambda/latest/dg/monitoring-logs.html |
| **CloudWatch Logs への送信** | https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs.html |

### CloudWatch Logs API

| トピック | URL |
|---------|-----|
| **PutLogEvents API** | https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutLogEvents.html |
| **CreateLogStream API** | https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_CreateLogStream.html |
| **ロググループとストリーム** | https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html |

### .NET ILogger

| トピック | URL |
|---------|-----|
| **ILogger インターフェース** | https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger |
| **ILoggerFactory インターフェース** | https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.iloggerfactory |
| **カスタムログプロバイダー作成** | https://learn.microsoft.com/en-us/dotnet/core/extensions/custom-logging-provider |

---

## 実装方法の比較

| 方法 | 複雑度 | パフォーマンス | 推奨度 |
|------|--------|--------------|--------|
| **CloudWatch Logs API 直接呼び出し** | ★★☆ | △ API呼び出しオーバーヘッド | ◎ |
| ILogger カスタムプロバイダー | ★★★ | △ 同上 | ○ |
| Lambda関数分離 | ★☆☆ | ◎ オーバーヘッドなし | △ |
| Subscription Filter で転送 | ★★☆ | ◎ | ○ |

---

## 推奨アーキテクチャ

複数ロググループへの出力が必要な場合、以下の構成を推奨：

```
┌─────────────────────────────────────────────────────────┐
│                    Lambda Function                       │
├─────────────────────────────────────────────────────────┤
│  Powertools Logger (標準出力)                            │
│      └── デフォルトロググループ (全ログ)                  │
│                                                         │
│  CloudWatch Logs API (追加出力)                         │
│      └── ERROR/WARN専用ロググループ (アラート用)         │
└─────────────────────────────────────────────────────────┘
```

または、**Subscription Filter を使ったサーバーレスルーティング**：

```
Lambda → CloudWatch Logs (単一) → Subscription Filter → 別ロググループ/Lambda/Firehose
```

---

## 注意事項

1. **API呼び出しコスト**: `PutLogEvents` は API 呼び出しのため、レイテンシが追加される
2. **エラーハンドリング**: `ResourceNotFoundException` 時のログストリーム作成処理が必要
3. **並行性**: 複数の Lambda インスタンスからの同時書き込みに対応が必要
4. **コスト**: 複数ロググループへの重複書き込みはコスト増加要因
