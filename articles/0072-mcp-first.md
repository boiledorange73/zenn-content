---
title: "初めて簡易MCPサーバーを作ってOpenAIエージェントとつないでみる"
emoji: "😀"
type: "tech"
topics: [OpenAI,MCP]
published: true
---

# はじめに

MCPをなんとしてもやりたくなったので、OpenAI API に課金もしていたので、OpenAIエージェント を使って MCP とつなげようと思い立ちました。

まずは Python から入れるところから始めないと…みたいなかんじでやってみます（さすがに前から入っていましたよ）。

# Pythonとかなんとか

まず、pip をアップグレードします。

```
python.exe -m pip install --upgrade pip
```

仮想環境としては venv を使いました。

```
python -m venv venv
```

仮想環境に入るには次を実行します (パスは適切なものに読み替えてください)

```
.\venv\Scripts\activate
```

venv環境にあっても pip をアップグレードします。

```
python.exe -m pip install --upgrade pip
```

必要なパッケージを導入します。

```
pip install dotenv "psycopg[binary]" fastmcp openai-agents
```

## メモ

次のことに注意してください。

- ``ImportError: no pq wrapper available.`` と出る時は libpq が無い場合。``psycopg`` でなく ``"psycopg[binary]"`` を入れるといいらしい。
- ``from agents import Agent, Runner`` とか出てくるからと言って ``pip install agents`` としてはいけません。TensorFlow が依存パッケージで、そのために Python を 3.11 に落とす必要があったりして、底なし沼に沈む道を進むことになります。

# コード

## srv1.py

MCPサーバーです。標準入出力専用です。ログ代わりにカレントフォルダに "srv1.log" というファイルに引数などを出力します。

参考: https://zenn.dev/karaage0703/articles/bc369a11a82263

```
from mcp.server.fastmcp import FastMCP
import sys

mcp = FastMCP("Demo", host="0.0.0.0", port=8000)

@mcp.tool()
def add(a: int, b: int) -> int:
    "Add two numbers"
    # ファイル出力
    with open("srv1.log", "a", encoding="utf-8") as f:
        f.write(f"add called: a={a}, b={b}\n")
        f.flush()
    # 結果を返す
    return a + b + 10
# ここからスタート
if __name__ == "__main__":
    mcp.run()
    # mcp.run(transport="streamable-http")
```

## cli1.py

srv1.pyを使用するOpenAIエージェントです。OpenAI API KEYを持っている場合に使えます。``dot_env``の読み込み元がカレントディレクトリの一つ上になっているので注意してください。

```
import asyncio
import os
from pathlib import Path

from agents import Agent, Runner
from agents.mcp import MCPServerStdio, create_static_tool_filter
from agents.model_settings import ModelSettings

from dotenv import load_dotenv
load_dotenv(dotenv_path="./../.env")

import sys # sys.executgable

async def main() -> None:
    # srv1.py（あなたの FastMCP サーバ）へのパス
    here = Path(__file__).resolve().parent
    server_py = here / "srv1.py"

    if not server_py.exists():
        raise FileNotFoundError(f"Could not find MCP server script: {server_py}")

    # stdio MCP server をサブプロセスとして起動して接続 :contentReference[oaicite:1]{index=1}
    async with MCPServerStdio(
        name="Demo",  # 任意（表示名）
        params={
            # "command": "python",
            "command": sys.executable,
            "args": [str(server_py)],
        },
        # add だけ見せる :contentReference[oaicite:2]{index=2}
        tool_filter=create_static_tool_filter(allowed_tool_names=["add"]),
        cache_tools_list=True,
    ) as mcp_server:
        agent = Agent(
            name="MCP Client Agent",
            # 指示
            instructions=(
                "You have access to an MCP tool named 'add'. "
                "Always use it for addition. "
                "Return only the final numeric answer."
            ),
            # 対象とする MCP サーバー一覧
            mcp_servers=[mcp_server],
            # ツール使用を強制したい場合
            model_settings=ModelSettings(tool_choice="required"),
        )
        # テスト: 7 + 22
        result = await Runner.run(agent, "7 + 21")
        print(result.final_output)
# ここからスタート
if __name__ == "__main__":
    if "OPENAI_API_KEY" not in os.environ:
        raise RuntimeError("OPENAI_API_KEY を環境変数に入れておいてください")
    asyncio.run(main())
```

# 実行してみる

``7+22``の結果が出力されます。

```
(venv) ...> cli1.py
38
```

わざと計算間違いをしてみていますが、そのまま騙されてくれています。

srv1.logは次のような出力が得られます。

```
add called: a=7, b=21
```

ということで、OpenAI のエージェントから実行できて、かつ MCP に騙されてくれたことが分かりました。

# おわりに

Pythonを入れるところから、OpenAI のエージェントから MCP を実行するところまでやってみました。

動くんですね、意外と簡単に。

