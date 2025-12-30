## Build Dependency Tools

> The following shows the correspondence between Cangjie components and the required build tools and versions. ✅ indicates that the component requires the build tool, while `--` indicates it does not.

### Linux Environment - Build tool and version requirements for each component:

| Build Tool Dependency          | cjc  | cjdb | runtime | std  | cjpm | cjfmt | cjlint | cjcov | cjtrace-recover | lsp  | hle  | stdx |
| ------------------------------ | ---- | ---- | ------- | ---- | ---- | ----- | ------ | ----- | --------------- | ---- | ---- | ---- |
| python: >3.7                   | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     |        |       | ✅               | ✅    | ✅    |      |
| googletest: >=1.18             | ✅    | --   | --      | --   | --   | --    | --     |       | ✅               | --   | --   |      |
| LLVM: >=15.0.4 and <16         | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅      |       | ✅               | --   | ✅    |      |
| binutils: >=2.38               | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅      |       | ✅               | --   | ✅    |      |
| gcc: >=7.3.0                   | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅      |       | ✅               | --   | ✅    |      |
| cmake: >=3.16.5 and <4         | ✅    | ✅    | ✅       | ✅    | --   | ✅     |        |       | ✅               | --   | ✅    |      |
| ninja: >1.10                   | ✅    | ✅    | ✅       | ✅    | --   | ✅     |        |       | ✅               | --   | ✅    |      |
| openssl: >3                    | --   | --   | --      | ✅    | --   | --    | --     |       |                 | --   | ✅    |      |
| swig: >4                       | --   | ✅    | --      | --   | --   | --    | --     |       |                 | --   | --   |      |

### macOS Environment - Build tool and version requirements for each component:

| Build Tool Dependency          | cjc  | cjdb | runtime | std  | cjpm | cjfmt | lsp  | hle  | stdx |
| ------------------------------ | ---- | ---- | ------- | ---- | ---- | ----- | ---- | ---- | ---- |
| python: >3.7                   | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     | ✅    | ✅    | ✅    |
| googletest: >=1.18             | ✅    | --   | --      | --   | --   | --    | ✅    | --   | --   |
| LLVM: >=16.0.0 and <17         | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| cmake: >=3.16.5 and <4         | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| ninja: >1.10                   | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| openssl: >3                    | --   | --   | --      | ✅    | --   | --    | --   | --   | ✅    |
| swig: >4                       | --   | ✅    | --      | --   | --   | --    | --   | --   | --   |

### Windows Environment - Build tool and version requirements for each component:

| Build Tool Dependency           | cjc  | cjdb | runtime | std  | cjpm | cjfmt | lsp  | hle  | stdx |
| ------------------------------- | ---- | ---- | ------- | ---- | ---- | ----- | ---- | ---- | ---- |
| python: >3.7                    | ✅    | ✅    | ✅       | ✅    | ✅    | ✅     | ✅    | ✅    | ✅    |
| googletest: >=1.18              | ✅    | --   | --      | --   | --   | --    | ✅    | --   | --   |
| LLVM: >=15.0.4 and <16          | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| binutils: >=2.38                | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| gcc: >=7.3.0                    | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| cmake: >=3.16.5 and <4          | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| ninja: >1.10                    | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| openssl: >3                     | --   | --   | --      | ✅    | --   | --    | --   | --   | ✅    |
| llvm-mingw-w64: 12.0.0          | ✅    | ✅    | ✅       | ✅    | --   | ✅     | ✅    | --   | ✅    |
| swig: >4                        | --   | ✅    | --      | --   | --   | --    | --   | --   | --   |

