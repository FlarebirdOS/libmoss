# Library Components

Relevant source files

* [dub.json](../dub.json)
* [meson\_options.txt](../meson_options.txt)
* [source/meson.build](../source/meson.build)

This document provides an overview of libmoss's modular component architecture and describes how the six main components interact and can be selectively enabled at build time. For detailed documentation of each individual component's functionality, see pages [3.1](3.1-moss-core) through [3.6](3.6-moss-fetcher).

## Component Architecture Overview

libmoss is designed as a modular library where only the core component is mandatory. All other components are optional and can be enabled or disabled at build time through Meson build options. This design allows consumers to build only the functionality they need, reducing binary size and compilation time.

The library consists of six primary components:

| Component | Directory | Mandatory | Build Option | Purpose |
| --- | --- | --- | --- | --- |
| moss-core | `source/moss/core/` | Yes | Always included | Foundational functionality required by all other components |
| moss-config | `source/moss/config/` | No | `--with-config` | Configuration management with layered YAML configuration |
| moss-db | `source/moss/db/` | No | `--with-db` | LMDB-based embedded database functionality |
| moss-deps | `source/moss/deps/` | No | `--with-deps` | Dependency resolution and tracking using xxHash |
| moss-format | `source/moss/format/` | No | `--with-format={binary,source,true,false}` | Binary and source format handling with compression |
| moss-fetcher | `source/moss/fetcher/` | No | `--with-fetcher={http,git,true,false}` | Data fetching via HTTP and Git protocols |

**Sources:** [meson\_options.txt1-5](../meson_options.txt#L1-L5) [source/meson.build1-39](../source/meson.build#L1-L39)

## Modular Component Selection

The following diagram illustrates how components are conditionally included based on build options:

```mermaid
flowchart TD

CORE["moss-core<br>(source/moss/core/)"]
CONFIG["moss-config<br>(source/moss/config/)"]
DB["moss-db<br>(source/moss/db/)"]
DEPS["moss-deps<br>(source/moss/deps/)"]
FORMAT["moss-format<br>(source/moss/format/)"]
FORMAT_BIN["Binary Format Support"]
FORMAT_SRC["Source Format Support"]
FETCHER["moss-fetcher<br>(source/moss/fetcher/)"]
FETCH_HTTP["HTTP Protocol Support"]
FETCH_GIT["Git Protocol Support"]
OPT_CONFIG["with-config<br>(boolean)"]
OPT_DB["with-db<br>(boolean)"]
OPT_DEPS["with-deps<br>(boolean)"]
OPT_FORMAT["with-format<br>(binary/source/true/false)"]
OPT_FETCHER["with-fetcher<br>(http/git/true/false)"]

OPT_CONFIG --> CONFIG
OPT_DB --> DB
OPT_DEPS --> DEPS
OPT_FORMAT --> FORMAT
OPT_FETCHER --> FETCHER

subgraph subGraph4 ["Build Options"]
    OPT_CONFIG
    OPT_DB
    OPT_DEPS
    OPT_FORMAT
    OPT_FETCHER
end

subgraph subGraph3 ["Optional Components"]
    CONFIG
    DB
    DEPS

subgraph moss-fetcher ["moss-fetcher"]
    FETCHER
    FETCH_HTTP
    FETCH_GIT
    FETCHER --> FETCH_HTTP
    FETCHER --> FETCH_GIT
end

subgraph moss-format ["moss-format"]
    FORMAT
    FORMAT_BIN
    FORMAT_SRC
    FORMAT --> FORMAT_BIN
    FORMAT --> FORMAT_SRC
end
end

subgraph subGraph0 ["Always Included"]
    CORE
end
```

**Sources:** [meson\_options.txt1-5](../meson_options.txt#L1-L5) [source/meson.build12-32](../source/meson.build#L12-L32)

## Build System Component Evaluation

The Meson build system evaluates component inclusion through a series of conditional checks. The following diagram shows the exact build variable names and control flow:

```mermaid
flowchart TD

START["Build Initiated"]
INIT["components = ['core']<br>(source/meson.build:12)"]
CHECK_CONFIG["with_moss_config<br>enabled?"]
ADD_CONFIG["components += 'config'<br>(source/meson.build:15)"]
CHECK_DB["with_moss_db<br>enabled?"]
ADD_DB["components += 'db'<br>(source/meson.build:19)"]
CHECK_DEPS["with_moss_deps<br>enabled?"]
ADD_DEPS["components += 'deps'<br>(source/meson.build:23)"]
CHECK_FETCHER["with_moss_fetcher_http<br>OR<br>with_moss_fetcher_git?"]
ADD_FETCHER["components += 'fetcher'<br>(source/meson.build:27)"]
CHECK_FORMAT["with_moss_format_binary<br>OR<br>with_moss_format_source?"]
ADD_FORMAT["components += 'format'<br>(source/meson.build:31)"]
ITERATE["foreach component : components<br>(source/meson.build:37)"]
SUBDIR["subdir('moss' / component)<br>(source/meson.build:38)"]

START --> INIT
INIT --> CHECK_CONFIG
CHECK_CONFIG --> ADD_CONFIG
CHECK_CONFIG --> CHECK_DB
ADD_CONFIG --> CHECK_DB
CHECK_DB --> ADD_DB
CHECK_DB --> CHECK_DEPS
ADD_DB --> CHECK_DEPS
CHECK_DEPS --> ADD_DEPS
CHECK_DEPS --> CHECK_FETCHER
ADD_DEPS --> CHECK_FETCHER
CHECK_FETCHER --> ADD_FETCHER
CHECK_FETCHER --> CHECK_FORMAT
ADD_FETCHER --> CHECK_FORMAT
CHECK_FORMAT --> ADD_FORMAT
CHECK_FORMAT --> ITERATE
ADD_FORMAT --> ITERATE
ITERATE --> SUBDIR
```

**Sources:** [source/meson.build12-39](../source/meson.build#L12-L39)

## Component Categories

### Mandatory Component

Only `moss-core` is unconditionally included in every build configuration. It is added to the `components` array at the beginning of the build process and provides foundational functionality required by all other components.

**Sources:** [source/meson.build12](../source/meson.build#L12-L12)

### Boolean Option Components

Three components use simple boolean flags for inclusion:

* **moss-config**: Enabled with `with-config` option (default: `true`)
* **moss-db**: Enabled with `with-db` option (default: `true`)
* **moss-deps**: Enabled with `with-deps` option (default: `true`)

**Sources:** [meson\_options.txt1-3](../meson_options.txt#L1-L3) [source/meson.build14-24](../source/meson.build#L14-L24)

### Combo Option Components

Two components provide sub-options for fine-grained control:

#### moss-format

The `with-format` option accepts multiple values:

* `'binary'`: Enable binary format support only
* `'source'`: Enable source format support only
* `'true'`: Enable all format support
* `'false'`: Disable format component entirely

Default value: `'true'`

The component is included if either `with_moss_format_binary` or `with_moss_format_source` evaluates to true.

**Sources:** [meson\_options.txt5](../meson_options.txt#L5-L5) [source/meson.build30-32](../source/meson.build#L30-L32)

#### moss-fetcher

The `with-fetcher` option accepts multiple values:

* `'http'`: Enable HTTP fetching only
* `'git'`: Enable Git fetching only
* `'true'`: Enable all fetcher support
* `'false'`: Disable fetcher component entirely

Default value: `'http'`

The component is included if either `with_moss_fetcher_http` or `with_moss_fetcher_git` evaluates to true.

**Sources:** [meson\_options.txt4](../meson_options.txt#L4-L4) [source/meson.build26-28](../source/meson.build#L26-L28)

## External Dependencies by Component

Different components require different external dependencies. The following table maps components to their primary external dependencies:

| Component | D Package Dependencies | System Library Dependencies | Purpose |
| --- | --- | --- | --- |
| moss-core | `tinyendian`, `elf-d` | - | Core data structures and ELF handling |
| moss-config | `dyaml` | - | YAML configuration parsing |
| moss-db | `lmdb-d` | `lmdb` | Embedded database operations |
| moss-deps | `xxhash-d` | `libxxhash` | Hash-based dependency tracking |
| moss-format | `zstd-d`, `xxhash-d` | `libzstd`, `libxxhash` | Compression and format integrity |
| moss-fetcher | `libgit2-d`, `ddbus` | `libcurl`, `libgit2` | Network and Git operations |

**Sources:** [dub.json14-45](../dub.json#L14-L45)

## Component Interdependencies

The following diagram shows the external dependencies required by each optional component, mapping to the actual dependency declarations:

```mermaid
flowchart TD

CORE["moss-core"]
CONFIG["moss-config"]
DB["moss-db"]
DEPS["moss-deps"]
FORMAT["moss-format"]
FETCHER["moss-fetcher"]
DYAML["dyaml<br>(vendor/dyaml)"]
TINYENDIAN["tinyendian<br>(vendor/tinyendian)"]
DDBUS["ddbus<br>(subprojects/ddbus)"]
LIBGIT2D["libgit2-d<br>(subprojects/libgit2-d)"]
LMBDD["lmdb-d<br>(subprojects/lmdb-d)"]
XXHASHD["xxhash-d<br>(subprojects/xxhash-d)"]
ELFD["elf-d<br>(vendor/elf-d)"]
ZSTDD["zstd-d<br>(subprojects/zstdoubledee)"]
LIBCURL["libcurl"]
LMDB["lmdb"]
LIBXXHASH["libxxhash"]
LIBZSTD["libzstd"]

CORE --> TINYENDIAN
CORE --> ELFD
CONFIG --> DYAML
DB --> LMBDD
DEPS --> XXHASHD
FORMAT --> ZSTDD
FORMAT --> XXHASHD
FETCHER --> LIBGIT2D
FETCHER --> DDBUS
LMBDD --> LMDB
XXHASHD --> LIBXXHASH
ZSTDD --> LIBZSTD
LIBGIT2D --> LIBCURL

subgraph subGraph2 ["System Libraries"]
    LIBCURL
    LMDB
    LIBXXHASH
    LIBZSTD
end

subgraph subGraph1 ["D Package Dependencies"]
    DYAML
    TINYENDIAN
    DDBUS
    LIBGIT2D
    LMBDD
    XXHASHD
    ELFD
    ZSTDD
end

subgraph Components ["Components"]
    CORE
    CONFIG
    DB
    DEPS
    FORMAT
    FETCHER
end
```

**Sources:** [dub.json14-45](../dub.json#L14-L45)

## Component Build Process

When the build system processes the components, it iterates through the `components` array and includes each component's subdirectory using `subdir('moss' / component)`. This means:

1. Core component directory `source/moss/core/` is always processed
2. Optional component directories are processed only if their conditions are met
3. Each component directory contains its own `meson.build` file defining compilation units

The iteration occurs at [source/meson.build37-39](../source/meson.build#L37-L39) using a foreach loop over the dynamically constructed `components` array.

**Sources:** [source/meson.build37-39](../source/meson.build#L37-L39)

## Component Documentation

Detailed documentation for each component is available in the following pages:

* **moss-core**: See [3.1](3.1-moss-core) for core functionality documentation
* **moss-config**: See [3.2](3.2-moss-config) for configuration management details
* **moss-db**: See [3.3](3.3-moss-db) for database component documentation
* **moss-deps**: See [3.4](3.4-moss-deps) for dependency management details
* **moss-format**: See [3.5](3.5-moss-format) for format handling documentation
* **moss-fetcher**: See [3.6](3.6-moss-fetcher) for fetcher component documentation

For information about building with specific component selections, see [2.2](2.2-component-selection) (Component Selection). For dependency details, see [6](6-external-dependencies) (External Dependencies).