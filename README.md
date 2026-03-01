# langchain-arcadedb

[![PyPI - Version](https://img.shields.io/pypi/v/langchain-arcadedb?label=%20)](https://pypi.org/project/langchain-arcadedb/#history)
[![PyPI - License](https://img.shields.io/pypi/l/langchain-arcadedb)](https://opensource.org/licenses/MIT)
[![PyPI - Downloads](https://img.shields.io/pepy/dt/langchain-arcadedb)](https://pypistats.org/packages/langchain-arcadedb)

This package contains the [LangChain](https://langchain.com) integration for
[ArcadeDB](https://arcadedb.com), the multi-model database with native graph,
document, key-value, time-series, and vector support.

## Installation

```bash
pip install langchain-arcadedb
```

This will also install the `neo4j` Python driver, which is used to connect to
ArcadeDB via its native Bolt protocol.

## Prerequisites

ArcadeDB must be running with the **Bolt protocol plugin** enabled.

### Docker (quickest way)

```bash
docker run --rm -p 2480:2480 -p 7687:7687 \
    -e JAVA_OPTS="-Darcadedb.server.plugins=Bolt:com.arcadedb.bolt.BoltProtocolPlugin" \
    -e arcadedb.server.rootPassword=playwithdata \
    arcadedata/arcadedb:latest
```

Then create a database (e.g. via the HTTP API):

```bash
curl -X POST http://localhost:2480/api/v1/server \
    -d '{"command":"create database mydb"}' \
    -u root:playwithdata \
    -H "Content-Type: application/json"
```

### Server / Embedded

If you run ArcadeDB from a distribution or embedded, add the Bolt plugin to your
server configuration:

```
arcadedb.server.plugins=Bolt:com.arcadedb.bolt.BoltProtocolPlugin
```

See the [ArcadeDB docs](https://docs.arcadedb.com/#Bolt) for details.

## Quick Start

```python
from langchain_arcadedb import ArcadeDBGraph

graph = ArcadeDBGraph(
    url="bolt://localhost:7687",
    username="root",
    password="playwithdata",
    database="mydb",
)

# Schema is auto-detected from existing data
print(graph.get_schema)

# Run Cypher queries
result = graph.query("MATCH (n:Person) RETURN n.name AS name")
```

## Use with GraphCypherQAChain

`ArcadeDBGraph` implements the `GraphStore` protocol, so it works as a drop-in
replacement wherever LangChain expects a graph store:

```python
from langchain_arcadedb import ArcadeDBGraph
from langchain_neo4j import GraphCypherQAChain
from langchain_openai import ChatOpenAI

graph = ArcadeDBGraph(
    url="bolt://localhost:7687",
    username="root",
    password="playwithdata",
    database="mydb",
)

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
chain = GraphCypherQAChain.from_llm(llm=llm, graph=graph, verbose=True)

answer = chain.invoke({"query": "Who does Alice know?"})
```

## Import Graph Documents

You can import structured graph documents directly:

```python
from langchain_arcadedb import ArcadeDBGraph, GraphDocument, Node, Relationship
from langchain_core.documents import Document

graph = ArcadeDBGraph(
    url="bolt://localhost:7687",
    username="root",
    password="playwithdata",
    database="mydb",
)

# Build a graph document
nodes = [
    Node(id="alice", type="Person", properties={"name": "Alice", "age": 30}),
    Node(id="bob", type="Person", properties={"name": "Bob", "age": 25}),
]
relationships = [
    Relationship(source=nodes[0], target=nodes[1], type="KNOWS"),
]
doc = GraphDocument(
    nodes=nodes,
    relationships=relationships,
    source=Document(page_content="Alice knows Bob"),
)

graph.add_graph_documents([doc])
```

## Configuration

Connection parameters can be passed directly or via environment variables:

| Parameter  | Environment Variable  | Default               |
|------------|-----------------------|-----------------------|
| `url`      | `ARCADEDB_URI`        | `bolt://localhost:7687` |
| `username` | `ARCADEDB_USERNAME`   | `root`                |
| `password` | `ARCADEDB_PASSWORD`   | `playwithdata`        |
| `database` | `ARCADEDB_DATABASE`   | (empty)               |

## Key Features

- **Bolt protocol** -- connects via the standard Neo4j Python driver
- **APOC-free** -- schema introspection and document import use pure Cypher
- **GraphStore protocol** -- works with `GraphCypherQAChain` out of the box
- **Graph document import** -- batched `MERGE` operations grouped by type
- **Lazy initialization** -- connection is established on first use
- **Schema auto-detection** -- node labels, relationship types, and property types are inferred from sampled data

## Development

```bash
# Install dependencies
uv sync --all-groups

# Run unit tests
make test

# Run integration tests (requires a running ArcadeDB instance)
make integration_tests

# Lint & format
make lint
make format
```

## Links

- [ArcadeDB Documentation](https://docs.arcadedb.com)
- [ArcadeDB on GitHub](https://github.com/ArcadeData/arcadedb)
- [LangChain Documentation](https://docs.langchain.com)
