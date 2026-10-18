---
title: .NET Framework
description: ".NET Framework 4.x (net48) runtime: where its classes behave differently from modern .NET or from the docs - gotchas and workarounds."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# .NET Framework

Classes of .NET Framework 4.x (net48, e.g. Office add-ins) that behave differently from modern .NET or from what their docs suggest. Read the matching file before relying on such a class in net48 code.

## Key facts

- `GZipStream` does not notice cut-off data and writes nothing for empty input: [gzipstream](gzipstream.md).

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [GZipStream in .NET Framework](gzipstream.md) — Read when net48 code packs or unpacks GZip (System.IO.Compression.GZipStream) and must detect damaged data or handle empty input.
<!-- END GENERATED CONTENTS -->
