# Claude SDK for Rust

[![Crates.io](https://img.shields.io/crates/v/claude-sdk.svg)](https://crates.io/crates/claude-sdk)
[![Documentation](https://docs.rs/claude-sdk/badge.svg)](https://docs.rs/claude-sdk)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A **complete, production-ready** Rust SDK for the Claude API with full support for server-side tools, structured output, and streaming. Built for [Colony Shell](https://github.com/mcfearsome/colony-shell) but usable standalone.

## What Makes This Special

- **Full API Parity** - Server tools, structured output, adaptive thinking, containers
- **Both Platforms** - Anthropic API + AWS Bedrock with unified interface
- **Production Ready** - 217 tests, retry logic, rate limit tracking, error handling
- **Type Safe** - `ToolDefinition` enum, `ContentBlock` variants, forward-compatible deserialization
- **Zero Cost** - Idiomatic async Rust with no runtime overhead

---

## Installation

```toml
[dependencies]
claude-sdk = "2.1"
tokio = { version = "1", features = ["full"] }

# Optional features
claude-sdk = { version = "2.1", features = ["bedrock", "repl"] }
```

**Features:**
- `anthropic` (default) - Anthropic API support
- `bedrock` - AWS Bedrock support
- `repl` - Interactive REPL binary
- `full` - All features enabled

---

## Quick Start

### Basic Chat

```rust
use claude_sdk::{ClaudeClient, Message, MessagesRequest, models};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = ClaudeClient::anthropic(std::env::var("ANTHROPIC_API_KEY")?);

    let request = MessagesRequest::new(
        models::CLAUDE_SONNET_4_5.anthropic_id,
        1024,
        vec![Message::user("Explain quantum entanglement")],
    );

    let response = client.send_message(request).await?;

    for content in &response.content {
        if let claude_sdk::ContentBlock::Text { text, .. } = content {
            println!("{}", text);
        }
    }

    Ok(())
}
```

### Server-Side Tools (Web Search + Code Execution)

```rust
use claude_sdk::{MessagesRequest, Message, ToolDefinition};
use claude_sdk::server_tools::{WebSearchTool, CodeExecutionTool};

let request = MessagesRequest::new(
    "claude-sonnet-4-5-20250929",
    4096,
    vec![Message::user("Search for the latest Rust release and calculate days since it")],
)
.with_tools(vec![
    ToolDefinition::from(WebSearchTool::new().with_max_uses(5)),
    ToolDefinition::from(CodeExecutionTool::new()),
]);
```

### Custom Tools

```rust
use claude_sdk::{CustomTool, ConversationBuilder, models};
use serde_json::json;

let tool = CustomTool::new(
    "get_weather",
    "Get current weather for a location",
    json!({
        "type": "object",
        "properties": {
            "location": {"type": "string"}
        },
        "required": ["location"]
    }),
)
.programmatic()   // No user confirmation needed
.with_strict();   // Strict JSON schema validation

let mut conversation = ConversationBuilder::new()
    .with_cached_system("You are a helpful assistant")
    .with_tool(tool);

conversation.add_user_message("What's the weather in Tokyo?");
let request = conversation.build(models::CLAUDE_SONNET_4_5.anthropic_id, 1024);
```

### Structured Output (Guaranteed JSON)

```rust
use claude_sdk::{MessagesRequest, Message};
use serde_json::json;

let request = MessagesRequest::new(
    "claude-sonnet-4-5-20250929",
    4096,
    vec![Message::user("Analyze this product review: ...")],
)
.with_json_schema(json!({
    "type": "object",
    "properties": {
        "sentiment": {"type": "string", "enum": ["positive", "negative", "mixed"]},
        "confidence": {"type": "number", "minimum": 0.0, "maximum": 1.0},
        "key_topics": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["sentiment", "confidence", "key_topics"]
}))
.with_adaptive_thinking();  // Claude decides how much reasoning to use
```

### Streaming Responses

```rust
use claude_sdk::{ClaudeClient, Message, MessagesRequest, StreamEvent};
use futures::StreamExt;

let client = ClaudeClient::anthropic(std::env::var("ANTHROPIC_API_KEY")?);

let request = MessagesRequest::new(
    "claude-sonnet-4-5-20250929",
    1024,
    vec![Message::user("Tell me a story")],
);

let mut stream = client.send_streaming(request).await?;

while let Some(event) = stream.next().await {
    match event? {
        StreamEvent::ContentBlockDelta { delta, .. } => {
            if let Some(text) = delta.text() {
                print!("{}", text);
            }
        }
        StreamEvent::MessageStop => break,
        _ => {}
    }
}
```

### AWS Bedrock

```rust
use claude_sdk::{ClaudeClient, MessagesRequest, Message, models};

// Uses AWS_PROFILE or default credential chain
let client = ClaudeClient::bedrock("us-east-1").await?;

let request = MessagesRequest::new(
    models::CLAUDE_SONNET_4_5.anthropic_id,  // Auto-converts to Bedrock format
    1024,
    vec![Message::user("Hello from Bedrock!")],
);

let response = client.send_message(request).await?;
```

### Extended Thinking

```rust
// Fixed budget
let request = MessagesRequest::new(
    "claude-sonnet-4-5-20250929", 16000,
    vec![Message::user("Prove the Pythagorean theorem")],
)
.with_thinking(10_000);  // 10K token budget

// Or adaptive (Claude decides)
let request = request.with_adaptive_thinking();
```

### Token Counting (Server-Side)

```rust
let client = ClaudeClient::anthropic(api_key);
let count = client.count_tokens(request).await?;
println!("Would use {} input tokens", count.input_tokens);
```

### Production Features

```rust
use claude_sdk::{MessagesRequest, Message, Metadata, ServiceTier, EffortLevel, CacheTtl};

let request = MessagesRequest::new("claude-sonnet-4-5-20250929", 4096,
    vec![Message::user("Analyze this data...")],
)
.with_metadata(Metadata { user_id: Some("user-123".into()) })
.with_service_tier(ServiceTier::Auto)        // Priority capacity if available
.with_effort(EffortLevel::High)              // Maximum quality
.with_inference_geo("us")                    // US region routing
.with_container("session-abc");              // Persistent code execution
```

---

## Examples

```bash
# Requires ANTHROPIC_API_KEY for API examples
cargo run --example simple_chat
cargo run --example streaming_chat
cargo run --example tool_use
cargo run --example prompt_caching

# No API key needed (prints request JSON)
cargo run --example server_tools
cargo run --example structured_output
```

---

## Supported Models

15 models with full metadata (context windows, pricing, Bedrock/Vertex IDs):

| Model | Context | Output | Thinking | Effort |
|-------|---------|--------|----------|--------|
| **Claude Fable 5** | 200K | 64K | Yes | No |
| **Claude Mythos 5** | 200K | 64K | Yes | No |
| **Claude Opus 4.8** | 200K | 64K | Yes | Yes |
| **Claude Opus 4.7** | 200K | 64K | Yes | Yes |
| **Claude Opus 4.6** | 200K | 64K | Yes | Yes |
| **Claude Sonnet 4.6** | 200K | 64K | Yes | No |
| **Claude Sonnet 4.5** | 200K | 64K | Yes | No |
| **Claude Haiku 4.5** | 200K | 64K | Yes | No |
| **Claude Opus 4.5** | 200K | 64K | Yes | Yes |
| Claude Opus 4.1 | 200K | 32K | Yes | No |
| Claude Sonnet 4 | 200K | 64K | Yes | No |
| Claude Opus 4 | 200K | 32K | Yes | No |
| + 3 legacy models | | | | |

---

## Complete API Coverage

**Messages API**
- Non-streaming & streaming (SSE)
- System prompts (cached with 5m/1h TTL)
- Multi-turn conversations
- Stop reasons: end_turn, max_tokens, stop_sequence, tool_use, pause_turn, refusal

**Tools**
- Custom tools with `CustomTool::new()` builder
- Server tools: web search, web fetch, code execution, bash, text editor
- Memory tool, tool search (BM25/regex)
- Tool choice (auto/any/tool/none with parallel control)
- Strict schema validation, deferred loading, eager input streaming

**Content Types**
- Text (with citations), Images (base64/URL/file_id), Documents (PDF/text)
- Search results (RAG), Thinking blocks (extended/redacted)
- Server tool use/results, Container uploads, Mid-conversation system blocks
- Forward-compatible: unknown block types captured as `ContentBlock::Unknown`

**Structured Output**
- JSON schema via `with_json_schema()` (guaranteed format)
- Adaptive thinking via `with_adaptive_thinking()`
- Effort levels: low, medium, high, xhigh, max

**Platform Support**
- Anthropic API (full, including token counting endpoint)
- AWS Bedrock (streaming + non-streaming, regional/global/us/eu/ap endpoints)

**Production**
- Retry with exponential backoff (rate limits + server errors)
- Rate limit header parsing (`RateLimitInfo`)
- Service tier routing (auto/standard_only)
- Metadata for abuse detection
- Geographic inference routing
- Container persistence for code execution
- Cache TTL control (5m/1h)
- Batch processing (100K requests, 50% discount)

---

## Project Structure

```
src/
├── lib.rs              # Public API & re-exports
├── client.rs           # HTTP client (Anthropic + Bedrock + token counting)
├── types.rs            # Request/response types, ContentBlock, ToolDefinition
├── server_tools.rs     # Server tool types (WebSearch, CodeExecution, etc.)
├── conversation.rs     # ConversationBuilder for multi-turn
├── models.rs           # Model registry (15 models)
├── streaming.rs        # SSE event types
├── error.rs            # Error taxonomy with is_retryable()
├── tokens.rs           # Local token counting (tiktoken-rs)
├── retry.rs            # Exponential backoff
├── files.rs            # Files API client
├── batch.rs            # Batch processing
├── prompts.rs          # System prompts (Claude Code, etc.)
├── structured.rs       # Structured output helpers
└── bin/
    ├── claude-repl.rs  # Interactive REPL
    └── update-changelog.rs
examples/
├── simple_chat.rs      # Basic non-streaming
├── streaming_chat.rs   # Real-time streaming
├── tool_use.rs         # Multi-turn with custom tools
├── prompt_caching.rs   # Cost optimization
├── server_tools.rs     # Web search + code execution
└── structured_output.rs # JSON schema + adaptive thinking
```

---

## Development

```bash
git clone https://github.com/mcfearsome/claude-agent-sdk-rust
cd claude-agent-sdk-rust
./scripts/install-hooks.sh

cargo test --all-features
cargo clippy --all-targets --all-features -- -D warnings
cargo fmt --all -- --check
```

---

## Status

**Version:** 2.1.0
**Tests:** 217 passing
**Platforms:** Anthropic + AWS Bedrock

---

## License

MIT License - See [LICENSE](LICENSE) for details
