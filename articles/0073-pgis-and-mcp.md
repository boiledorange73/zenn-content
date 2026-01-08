---
title: "MCP+PostGISで緯度経度に強いエージェントができるのです（言い過ぎ）"
emoji: "😀"
type: "tech"
topics: [PostGIS,GIS,OpenAI,Claude,MCP]
published: true
---

# はじめに

MCPをやりたいのですが、手持ちに Claude デスクトップもなく、でも OpenAI API にストックのお金があったので、とりあえずこれで遊んでみようと思った次第。

# データベースとその設定

データベースは ``loaclhost`` の ``mncpl`` です。そこに ``raw_mncpl`` というテーブルをおいています。

```
% ogr2ogr -f PGDUMP --config PG_USE_COPY YES \
  -lco SPATIAL_INDEX=GIST -lco GEOMETRY_NAME=geom -lco FID=gid \
  -nln raw_mncpl raw_mncpl.sql N03-20250101.geojson

% createdb mncpl
% psql -d mncpl -c "CREATE EXTENSION postgis_topology cascade"
% psql -d mncpl -f raw_mncpl.sql
```

今回は``www``に対して``raw_mncpl``を``SELECT``する権限を``GRANT``しています。

```
GRANT SELECT ON raw_mncpl TO www;
```

あと、むかしの PostGIS では ``spatial_ref_sys`` の ``GRANT`` も必要でしたが、今はどうも ``PUBLIC`` に ``GRANT`` されています。

# コード

## srv2.py

MCPサーバーです。標準入出力で動かすことも、HTTPで動かすこともできます。

切り替えは ``mcp.run()`` の引数です。``mcp.run()`` と引数なしにすると標準入出力で動き、``mcp.run(transport="streamable-http")`` とすると HTTP で動かします。

```
from mcp.server.fastmcp import FastMCP

import psycopg
from psycopg.rows import dict_row

import sys
import logging

# DSNは適切に変更してください
DSN="dbname=mncpl host=localhost user=www"

# 参考: https://qiita.com/engchina/items/3a1c412b21c1ae63b7d1
mcp = FastMCP("Demo", host="0.0.0.0", port=8000)

@mcp.tool()
def add(a: int, b: int) -> int:
    print(f"add called: {a}, {b}")
    return a + b

@mcp.tool()
def find_pname(lon: float, lat: float) -> str:
    """
    Finds the prefecture name from the given longitude and latitude.
    
    Parameters:
      + lon: float - Longitude in degrees (EPSG:6668 / JGD2011).
      + lat: float - Latitude in degrees (EPSG:6668 / JGD2011).
    Notes
    -----
    - Input coordinates MUST be JGD2011 geographic coordinates (EPSG:6668).
    - The order of arguments is longitude, followed by latitude.
    """
    logger = logging.getLogger(__name__)
    logger.info("find_pname(%f,%f)" % (lon,lat))
    with psycopg.connect(
        DSN,
        row_factory=dict_row,
    ) as conn, conn.cursor() as cur:
        query = """
          SELECT n03_007 mcode, n03_001 pname, coalesce(n03_003,'') || coalesce(n03_004, '') || coalesce(n03_005,'') mname FROM raw_mncpl
          WHERE ST_Intersects(geom, ST_SetSRID(ST_Point(%s, %s), 6668))
          LIMIT 1;
        """
        cur.execute(query, (lon, lat))
        row = cur.fetchone()
        if row:
            logger.info("found %s" % (row["pname"]))
            return row["pname"]
        logger.info("not found.")
        return None

@mcp.tool()
def find_mname(lon: float, lat: float) -> str:
    """
    Finds the municipality name from the given longitude and latitude.
    
    Parameters:
      + lon: float - Longitude in degrees (EPSG:6668 / JGD2011).
      + lat: float - Latitude in degrees (EPSG:6668 / JGD2011).
    Notes
    -----
    - Input coordinates MUST be JGD2011 geographic coordinates (EPSG:6668).
    - The order of arguments is longitude, followed by latitude.
    """
    logger = logging.getLogger(__name__)
    logger.info("find_mname(%f,%f)" % (lon,lat))
    with psycopg.connect(
        DSN,
        row_factory=dict_row,
    ) as conn, conn.cursor() as cur:
        query = """
          SELECT n03_007 mcode, n03_001 pname, coalesce(n03_003,'') || coalesce(n03_004, '') || coalesce(n03_005,'') mname FROM raw_mncpl
          WHERE ST_Intersects(geom, ST_SetSRID(ST_Point(%s, %s), 6668))
          LIMIT 1;
        """
        cur.execute(query, (lon, lat))
        row = cur.fetchone()
        if row:
            logger.info("found %s" % (row["mname"]))
            return row["mname"]
        logger.info("not found.")
        return None

@mcp.tool()
def find_mcode(lon: float, lat: float) -> str:
    """
    Finds the municipality code from the given longitude and latitude.
    
    Parameters:
      + lon: float - Longitude in degrees (EPSG:6668 / JGD2011).
      + lat: float - Latitude in degrees (EPSG:6668 / JGD2011).
    Notes
    -----
    - Input coordinates MUST be JGD2011 geographic coordinates (EPSG:6668).
    - The order of arguments is longitude, followed by latitude.
    """
    logger = logging.getLogger(__name__)
    logger.info("find_mcode(%f,%f)" % (lon,lat))
    with psycopg.connect(
        "dbname=mncpl host=localhost user=www",
        row_factory=dict_row,
    ) as conn, conn.cursor() as cur:
        query = """
          SELECT n03_007 mcode, n03_001 pname, coalesce(n03_003,'') || coalesce(n03_004, '') || coalesce(n03_005,'') mname FROM raw_mncpl
          WHERE ST_Intersects(geom, ST_SetSRID(ST_Point(%s, %s), 6668))
          LIMIT 1;
        """
        cur.execute(query, (lon, lat))
        row = cur.fetchone()
        if row:
            logger.info("found %s" % (row["mcode"]))
            return row["mcode"]
        logger.info("not found.")
        return None

def test():
    print(find_pname(139.769954, 35.687567))
    print(find_mname(139.769954, 35.687567))
    print(find_mcode(139.769954, 35.687567))

if __name__ == "__main__":
    # mcp.run(transport="streamable-http")
    mcp.run()
    #test()
```

## cli2.py (Claudeデスクトップでは不要)

srv2.pyを使用するOpenAIエージェントです。OpenAI API KEYを持っている場合に使えます。``dot_env``の読み込み元がカレントディレクトリの一つ上になっているので注意してください。

```
import os

from dotenv import load_dotenv

load_dotenv(dotenv_path="./../.env")

from pathlib import Path
import sys

import asyncio
from agents import Agent, Runner
from agents.mcp import MCPServerStreamableHttp, MCPServerStdio, create_static_tool_filter
from agents.model_settings import ModelSettings

# ツール名一覧
ALLOWED_TOOL_NAMES = ["find_pname", "find_mname", "find_mcode"]

# http, stdio の双方の共通部
async def fn_core(mcp_server):
    # 接続確認 tool の一覧を得る
    tools = await mcp_server.list_tools()
    print("MCP tools:", [t.name for t in tools])
    # エージェント生成
    agent = Agent(
        name="MCP Client Agent",
        instructions="""
          Use the MCP tool `find_pref` to get the prefecture name from the given longitude and latitude.
          Use `find_mname` to get the municipality name from the given longitude and latitude.
          Use `find_mcode` to get the municipality code (not the name) from the given longitude and latitude.
        """,
        mcp_servers=[mcp_server],
        # tool_choice: "required": 必ず tool 呼ぶ、"auto": エージェント判断、"none": 常に呼ばない。
        model_settings=ModelSettings(tool_choice="auto"),
    )
    # プロンプトを投げて、結果を出力
    result = await Runner.run(agent, "緯度経度が(35, 135)の地点の都道府県名と県庁所在地を教えて。")
    print(result.final_output)

# httpで動いているサーバにアクセスする場合
async def main_remote() -> None:
    url = "http://127.0.0.1:8000/mcp"
    # Streamable-HTTP MCP server をサブプロセスとして起動して接続
    async with MCPServerStreamableHttp(
        name="Demo (FastMCP HTTP)",
        params={
            "url": url,
            # 必要ならヘッダ等もここに
            # "headers": {"Authorization": "Bearer ..."},
            "timeout": 10,
        },
        tool_filter=create_static_tool_filter(allowed_tool_names=ALLOWED_TOOL_NAMES),
        cache_tools_list=True,
        max_retry_attempts=3,
    ) as mcp_server:
        await fn_core(mcp_server)

# stdioでサーバを走らせる場合
async def main_stdio() -> None:
    # srv1.py（あなたの FastMCP サーバ）へのパス
    here = Path(__file__).resolve().parent
    server_py = here / "srv2.py"
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
        tool_filter=create_static_tool_filter(allowed_tool_names=ALLOWED_TOOL_NAMES),
        cache_tools_list=True,
    ) as mcp_server:
        await fn_core(mcp_server)


# 実行開始時
if __name__ == "__main__":
    if "OPENAI_API_KEY" not in os.environ:
        raise RuntimeError("Please set OPENAI_API_KEY environment variable.")
    # asyncio.run(main_remote())
    asyncio.run(main_stdio())
```


# Claude デスクトップで MCP を使ってみよう

``%APPDATA%\Claude\claude_desktop_config.json`` をいじることで MCP が使えます。

もしくはアプリ右下のアカウント名をクリックすると出る「設定」で、「開発者」カテゴリ（一番下）をクリックして「設定を編集」をクリックしても出ます。

## 標準入出力版

``srv2.py``を stdio で実行する場合には、次のようにします。

```
{
  "mcpServers": {
    "demo": {
      "command": "C:\\path_to_venv\\Scripts\\python.exe",
      "args": ["C:\\path_to_script\\srv2.py"]
    }
  }
}
```

venvを使っている場合は、**venv内の ``python`` でないと導入パッケージが入っていません**。

## HTTP版

``srv2.py``を HTTP で実行している場合には、次のようにします。

ただし**Node.jsが必要です**。

```
{
  "mcpServers": {
    "demo": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "http://127.0.0.1:8000/mcp"
      ]
    }
  }
}
```

## 設定変更後は完全再起動を

書換後、完全再起動が必要です。

まず完全に終了させるために、タスクトレイに出ている Claude のアイコン（ウニみたいなの）を右クリックして「終了」をクリックします。その後 Claude を通常起動させます。

# MCP援用の結果

経度緯度(135 35) <https://maps.gsi.go.jp/#16/35/135/> について質問したらどうなるか見てみましょう。

## OpenAIエージェント+MCPの場合

「緯度経度が(35, 135)の地点の都道府県名と県庁所在地を教えて。」に対して次のような結果を得ました。

```
緯度経度 (35, 135) の地点は「兵庫県西脇市」に位置します。

兵庫県の県庁所在地は「神戸市」です。
```

## Claude+MCPの場合

「緯度経度が(35, 135)の地点の都道府県名と県庁所在地を教えて。」に対して次のような結果を得ました。

```
緯度経度(35, 135)の地点は兵庫県にあります。
この地点は具体的には兵庫県西脇市に位置しています。
兵庫県の県庁所在地は神戸市です。
```

# 単独の結果

2026年1月7日に試しました。

## ChatGPT 5.2単独

「緯度経度が(35, 135)の地点の都道府県名と県庁所在地を教えて。」に対して次のような結果を得ました。

```
緯度経度 (35, 135) は、日本の 近畿地方 に位置し、以下の地点に該当します。
都道府県名：京都府
県庁所在地：京都市
補足：この座標は京都市中心部付近（概ね市街地）にあたります。
```

京都じゃないよ。

## Claude単独

「緯度経度が(35, 135)の地点の都道府県名と県庁所在地を教えて。」に対して次のような結果を得ました。

```
緯度経度が(35, 135)の地点、つまり北緯35度・東経135度の地点は兵庫県西脇市 にあります。この地点には「日本のへそ公園」 があることで知られています。
都道府県名: 兵庫県
県庁所在地: 神戸市
なお、東経135度は日本標準時を決める基準となる子午線で、兵庫県明石市を通っています。北緯35度と東経135度が交わる西脇市は、地理上の「日本の中心」とも呼ばれています。
```

Claudeだと問題ないのではないか…。

# なんとしても優位なところを見るまで終われない

なんとしても Claude さん単独では間違えてもらわないといけない。ということで、なんとしても間違えてもらうこととしました。

(139.769954, 35.687567) <https://maps.gsi.go.jp/#17/35.687567/139.769954/> の地点の市区町村名を教えてもらいます。



## Claude+MCP

「緯度経度が(139.769954, 35.687567)の地点の市区町村名を教えて。」

```
この地点は中央区です。
```

## Claude単独 (Webインターフェース)

「緯度経度が(139.769954, 35.687567)の地点の市区町村名を教えて。」

```
緯度経度35.687567, 139.769954は、以前の検索結果（千代田区の医療機関データ）から、東京都千代田区に該当する座標であることが確認できました。より詳しくは、この座標は千代田区の永田町周辺に位置していると推測されます。
東京都千代田区
が、緯度35.687567、経度139.769954の地点の市区町村名です。
```

# おわりに

PostGISを使ったMCPサーバーの標準出力版とHTTP版とを（同一ファイル内に）作りました。このサーバーは、緯度経度から都道府県名を出すツール、緯度経度から市区町村名を出すツール、緯度経度から自治体コードを出すツール、を持っています。

また、OpenAIエージェントだけでなく、Claudeデスクトップからも実行してみました。

そして MCP の実行結果から、座標値はおそらく生成AIの弱いところだろうと思って、MCPを援用することでおかしな回答を出すことが抑制できました…と言い切っていいのかどうか、というか、正当な評価ができてるのかどうか分からないです。なんじゃそれ。

