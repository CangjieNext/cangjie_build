## Build Dependency Tools

### Linux Environment: Build tools and version requirements for each component are as follows:

| Build Dependency Tool       | cjc  | cjdb | runtime | std  | cjpm | cjfmt | lsp  | hle  | stdx |
|-----------------------------| ---- | ---- | ------- | ---- | ---- | ----- | ---- | ---- | ---- |
| python: >3.7                | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     | ✅    | ✅    | ✅    |
| googletest: >=1.18          | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     | ✅    | ✅    | ✅    |
| llvm: >=15.0.4 and <16      | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| binutils: >=2.38            | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| gcc: >7.3.0                 | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| cmake: >=3.16.5 and <4      | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| ninja: >1.10                | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| openssl: >3                 | ❌    | ❌    | ❌       | ✅    | ❌    | ❌     | ❌    | ❌    | ✅    |

### macOS Environment: Build tools and version requirements for each component are as follows:

| Build Dependency Tool       | cjc  | cjdb | runtime | std  | cjpm | cjfmt | lsp  | hle  | stdx |
|-----------------------------| ---- | ---- | ------- | ---- | ---- | ----- | ---- | ---- | ---- |
| python: >3.7                | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     | ✅    | ✅    | ✅    |
| googletest: >=1.18          | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     | ✅    | ✅    | ✅    |
| m4: >1                      | ✅    | ❌    | ❌       | ❌    | ❌    | ❌     | ❌    | ❌    | ❌    |
| llvm: >=16.0.0 and <17      | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| cmake: >=3.16.5 and <4      | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| ninja: >1.10                | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| openssl: >3                 | ❌    | ❌    | ❌       | ✅    | ❌    | ❌     | ❌    | ❌    | ✅    |

### Windows Environment: Build tools and version requirements for each component are as follows:

| Build Dependency Tool       | cjc  | cjdb | runtime | std  | cjpm | cjfmt | lsp  | hle  | stdx |
|-----------------------------| ---- | ---- | ------- | ---- | ---- | ----- | ---- | ---- | ---- |
| python: >3.7                | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     | ✅    | ✅    | ✅    |
| googletest: >=1.18          | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     | ✅    | ✅    | ✅    |
| llvm: >=15.0.4 and <16      | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| binutils: >=2.38            | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| gcc: >7.3.0                 | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| cmake: >=3.16.5 and <4      | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| ninja: >1.10                | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |
| openssl: >3                 | ❌    | ❌    | ❌       | ✅    | ❌    | ❌     | ❌    | ❌    | ✅    |
| mingw-w64: >=11.0.1         | ✅    | ✅    | ✅       | ✅    | ❌    | ✅     | ✅    | ❌    | ✅    |