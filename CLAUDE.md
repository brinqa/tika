# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Apache Tika is a toolkit for detecting and extracting metadata and structured text content from various document formats. Version 3.2.3, targeting Java 8 (`maven.compiler.release=8`). Uses Maven 3 build system.

## Build Commands

```bash
# Full build (all modules)
mvn clean install

# Build a specific module with dependencies
mvn clean install -am -pl :tika-server-standard

# Skip OSS vulnerability checks (if a dependency CVE causes build failure)
mvn clean install -Dossindex.skip

# Run a single test class
mvn test -pl :tika-parser-pdf-module -Dtest=PDFParserTest

# Run a single test method
mvn test -pl :tika-parser-pdf-module -Dtest=PDFParserTest#testMethodName

# Skip a failing test
mvn clean install -Dtest=\!UnpackerResourceTest#testPDFImages

# Integration tests require Docker (skipped automatically if Docker is unavailable)
```

Surefire runs with `-Xmx4g -Djava.awt.headless=true`. Tests use JUnit 5.

## Architecture

### Core Interfaces (tika-core)

The three foundational interfaces:

- **`Parser`** — `parse(InputStream, ContentHandler, Metadata, ParseContext)`. Parsers emit SAX events (XHTML) to a `ContentHandler`. Registered via Java SPI (`META-INF/services/org.apache.tika.parser.Parser`).
- **`Detector`** — `detect(InputStream, Metadata)` returns a `MediaType`. Also SPI-registered.
- **`ContentHandler`** — SAX handler for parser output. Key implementations: `BodyContentHandler` (text), `ToXMLContentHandler` (XHTML).

Key composition classes:
- `AutoDetectParser` — detects MIME type then delegates to the appropriate parser
- `CompositeParser` / `CompositeDetector` — delegate to component implementations by media type
- `DefaultParser` / `DefaultDetector` — auto-discover implementations via ServiceLoader
- `RecursiveParserWrapper` — extracts metadata from embedded documents recursively

Configuration: `TikaConfig` loads XML config defining parser/detector composition. Parsers support `@Field` annotation for XML-based property injection and `Initializable` interface for post-construction setup.

### Module Structure

**tika-parsers** — Three tiers based on dependency weight:
- `tika-parsers-standard` — Core parsers with no network/native dependencies. Contains 22 individual modules under `tika-parsers-standard-modules/` (e.g., `tika-parser-pdf-module`, `tika-parser-microsoft-module`, `tika-parser-html-module`). Assembled into `tika-parsers-standard-package`.
- `tika-parsers-extended` — Parsers requiring network access or native code
- `tika-parsers-ml` — Machine learning parsers (large dependencies like DL4J)

**tika-server** — REST API via Apache CXF (JAX-RS):
- `tika-server-core` — JAX-RS resources, config, server lifecycle
- `tika-server-standard` — Bundles standard parsers into deployable server
- `tika-server-client` — Java client library

**tika-pipes** — Batch processing pipeline:
- `tika-pipes-iterators` — Source iterators (CSV, JSON, JDBC, S3, Kafka, Solr, GCS, Azure)
- `tika-fetchers` — Document fetching from various backends
- `tika-emitters` — Output to backends (S3, Solr, OpenSearch, Kafka, filesystem)
- `tika-pipes-reporters` — Progress reporting
- `PipesServer` runs parsing in forked subprocesses for isolation

**Other modules**: `tika-app` (standalone CLI), `tika-batch` (batch processing), `tika-grpc` (gRPC server), `tika-langdetect` (language detection with multiple backends), `tika-eval` (evaluation tools), `tika-handlers` (content extraction handlers), `tika-integration-tests` (Docker-based integration tests).

### SPI Plugin Pattern

All major extension points use `META-INF/services/` for discovery. Each parser module maintains its own service file. To add a new parser:
1. Implement `Parser` interface
2. Register in `META-INF/services/org.apache.tika.parser.Parser`
3. The `DefaultParser` / `AutoDetectParser` will auto-discover it

### Testing

Base test class: `org.apache.tika.TikaTest` (in tika-core) provides helper methods for parsing test documents and asserting content/metadata. Test resources go in `src/test/resources/`. Parser tests typically extend `TikaTest` and use `AUTO_DETECT_PARSER` or a specific parser instance.

### Key Dependencies

PDFBox 3.0.5 (PDF), POI 5.4.1 (Office), CXF 4.0.9 (REST), Jetty 11.0.26 (HTTP server), Lucene 9.12.2 (text analysis). Several dependency versions are capped due to JDK 8 compatibility (e.g., CXF 4.1+ needs JDK 17, Spring 6 needs JDK 17).

## Branch Notes

The `main` branch is the primary development branch. PRs should target `main`. The `master` branch is locked and unused since September 2020.
