<!-- language-policy:start -->
# 全局语言要求

- 始终使用简体中文与用户交流。
- 除非用户明确要求英文，否则所有解释、总结、计划、问题、错误分析都使用中文。
- 代码、命令、文件名、API 名称、库名、函数名、类型名、日志和原始错误信息保持原文。
- 如果外部资料、文档或错误信息是英文，需要用中文解释和总结。
- 生成代码注释时，优先使用中文，除非项目已有明确的英文注释风格。

<!-- language-policy:end -->


<!-- codebase-memory-mcp:start -->
# Codebase Knowledge Graph (codebase-memory-mcp)

This project uses codebase-memory-mcp to maintain a knowledge graph of the codebase.
ALWAYS prefer MCP graph tools over grep/glob/file-search for code discovery.

## Priority Order
1. `search_graph` — find functions, classes, routes, variables by pattern
2. `trace_path` — trace who calls a function or what it calls
3. `get_code_snippet` — read specific function/class source code
4. `query_graph` — run Cypher queries for complex patterns
5. `get_architecture` — high-level project summary

## When to fall back to grep/glob
- Searching for string literals, error messages, config values
- Searching non-code files (Dockerfiles, shell scripts, configs)
- When MCP tools return insufficient results

## Examples
- Find a handler: `search_graph(name_pattern=".*OrderHandler.*")`
- Who calls it: `trace_path(function_name="OrderHandler", direction="inbound")`
- Read source: `get_code_snippet(qualified_name="pkg/orders.OrderHandler")`
<!-- codebase-memory-mcp:end -->
