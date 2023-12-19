---
title: "Reachability Analysis With Tree Sitter in Rust — Part 1"
date: 2023-12-19T11:29:06+08:00
draft: false
---

> **NOTE:** An early implementation of a project to implement the techniques described in this post can be found [here](https://github.com/kwekmh/code-analyser).

For those involved in vulnerability management, you will understanding how challenging it can be to triage the multitude of vulnerabilities that are raised by a variety of scanners. Some of them, despite their low severity, can be critical to a particular codebase due to how the affected functions are being used. Some of the higher-severity ones may be irrelevant. In this post, I am trying to address the noise from dependency scanners, particularly for Node.js and NPM. A single Node.js or frontend project using a popular framework such as Next.js can have millions of dependencies. There can also be infinite layers of nested dependencies (OK, I exaggerate, but I am sure some of you appreciate the jest!). One of the first steps I take when I triage and assess a vulnerability that a dependency scanner has raised is to understand how the package is being used, and if the vulnerable function is called. Of course, as far as possible, the recommendation is to upgrade the package regardless! However, sometimes that is not possible, either because there are no fixed versions yet, or there are dependencies that rely on a particular vulnerable version of a package that the developers cannot easily remove.

It would be much easier if some of this work could be automated. The concept of **Reachability Analysis** comes to mind. For those of you who are into compiler theory, you may be reminded of **live variable analysis** and **dead code elimination**. The concepts are similar; the idea is to identify how a vulnerable function is being used in a particular project, if at all. This is basically my first step of triaging which can take significant time and effort, and automating this away would help to eliminate low-hanging fruits.

I have always been interested in compiler theory. I remember reading the [Dragon Book](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools) in my formative years, getting all excited about lexical analysis, parsing, code generation and optimisation techniques. I implemented a [recursive descent parser](https://en.wikipedia.org/wiki/Recursive_descent_parser) for a subset of the C++ programming language (I cannot remember which standard it was for; it may have been C++11) that was used in the backend for a code obfuscator. I also implemented recursive descent parsers for toy language in my programming courses. I think one of them was written in Haskell. Why do I mention parsing now? My approach to **reachability analysis** was to generate parse trees or AST trees of the source code in a project, and attempt to use that to identify if and how vulnerable functions were being used. The parsing, or compiler frontend, is typically the easiest part of compilers, but it is also an indispensable first step.

I could build a recursive descent parser from scratch, annotating the nodes as I traversed the trees, but I decided that there were better options than reinventing the wheel. [Tree-sitter](https://tree-sitter.github.io/tree-sitter/) is a popular tool and library that is used to generate parsers and perform incremental parsing. It can generates parse trees that are represented in the form of [s-expressions](https://en.wikipedia.org/wiki/S-expression), which, as someone who attempted to pick up Lisp at some point in time in the past, I was extremely enthralled by. More importantly, it had ready-made parser libraries for the popular languages, including **TypeScript**, which was what I chose to start with.

As a budding Rustacean, what better programming language is there than Rust to implement this in? Thankfully, `tree-sitter` has Rust bindings, so that helped to cement the decision. For this post, I will only focus on some parts of that parsing that I have implemented overnight. The tool itself to perform reachability analysis is incomplete, although I will be working on it over the next few days or weeks, and I hope that it will reach a usable state.

Documentation for `tree-sitter` was somewhere between great and mediocre. I did manage to get started relatively quickly, but writing queries (in an `s-expression`-inspired DSL) rather than traversing the tree was rather challenging, and I had to refer to Google for help as `tree-sitter`'s documentation was often insufficient for my needs.

As it is, parsers typically operate on single files, and interestingly enough, they do not seem to come out of the bos with helper functions to parse an entire directory. I had to write helper functions to iterate recursively through a specified directory, identify the files with the correct extensions, and stored the parsed trees into a `HashMap` that mapped the trees back to the file paths. The abstracted version looks like this:

```
let paths = find_by_extensions_in_dir(&OsStr::new("C:\\Users\\Ming\\projects\\typescript-ast-generator"), &vec![OsStr::new("ts")]);

let parsed = parse_directory(&mut parser, paths);
```

The extensions are provided in a `Vec`, because it is possible that there are multiple extensions that could refer to the same type of source files. I used `OsStr` because there were no guarantees that a system's paths were represented as Unicode strings, which Rust strings are, so it was better to use `OsStr` instead of risking data corruption by attempting to represent path-related strings as Rust strings.

Where did `parser` come from? That was trivial to initialise.

```
let mut parser = Parser::new();
parser.set_language(tree_sitter_typescript::language_typescript()).expect("Error loading TypeScript grammar");
```

Of course, you need the right dependencies in your `Cargo.toml`:

```
[dependencies]
tree-sitter = "0.20.10"
tree-sitter-typescript = "0.20.3"

[build-dependencies]
cc="*"
```

I will not go into details of the setup here, as the documentation for `tree-sitter` will do better job than I can. I will describe two `tree-sitter` queries that I wrote to identify the two most interesting aspects of a TypeScript source file that are relevant to reachability analysis:

1. Function and method calls
2. Package imports

These are the two pieces of information that will be useful in helping me to assess how vulnerable functions are being used. Of course, there will potentially be more nodes that I will need to extract from the parse trees before I can correlate them and the vulnerability information, but this is a start.

How do you craft a `tree-sitter` query in Rust? There are two parts:

1. First, you create a `Query` object.
2. Secondly, you create a `QueryCursor` object that does the actual heavy lifting, and returns the matches and captures from your query.

In essence, it is something like:

```
let query = Query::new(tree_sitter_typescript::language_typescript(),
                    r#"
                    (call_expression ...)
                    "#).unwrap();
let mut query_cursor = QueryCursor::new();
let all_matches = query_cursor.matches(&query, root_node, source.as_bytes());                    
```

As the parse trees does not include names or strings (the trees only describe the syntax), the optional parameter `source.as_bytes()` is important if your query involves matching based on names (e.g. to find all function calls that have a particular function name).

You can then iterate through the match and capture groups:

```
for each_match in all_matches {
    for capture in each_match.captures {
        ...
    }
}
```

The above is the high-level flow of how using `tree-sitter` to parse source code and write queries to identify particular nodes works. To summarise:

1. Create a `tree-sitter` parser object.
2. Parse the source code with the parser.
3. Create a `Query` object.
4. Create a `QueryCursor` object.
5. Using the `QueryCursor` object, you can generate matches from the query, with the root node of the tree (or the node that you want to start with), and the original source code (for certain types of queries).

This was the easy part. The challenging part was in crafting the queries. The [tree-sitter playground](https://tree-sitter.github.io/tree-sitter/playground) was useful in allowing me to test my queries, but it also lacked the ability to distinguish between the match and capture groups, so I had to debug those by running my program and printing debug lines.

I started with the query to find all function and method calls of a specified name as it felt easier:

```
let query = Query::new(tree_sitter_typescript::language_typescript(),
               &format!(r#"
                ((call_expression
                  function: [
                    (identifier) @function
                      (member_expression
                        property: (property_identifier) @method
                      )]
                 (#eq? @function "{}")
                 (#eq? @method "{}")
                ))"#, function_name, function_name)).unwrap();
```

Yes, you saw that right! You can use `format!` and format specifiers with raw strings. In fact, `{}` will need to be escaped as `{{}}` if you want to represent it literally. In this case, I am trying to identify all `call_expression`s that have a specific identifier for the call name, regardless of whether they are function calls (e.g. function()) or method calls (e.g. obj.function()). This was straightforward to write and test.

I then followed up with attempting to identify all imports, and to break them up into the correct match groups. For example, if I have a statement `import * as a from b`, I will need to capture `a` and `b` individually as I need to identify what the imported name is and what the package name is, but I will need them to be in the same match group so I know they are related. The grammar for imports is also more convoluted, as it can involve string-based names and aliases. This was what I came up with:

```
let query = Query::new(tree_sitter_typescript::language_typescript(),
                        r#"
                        (import_statement
                        "import"
                        (import_clause ((identifier)? @named_import
                                (namespace_import (identifier) @namespace_import)?
                                            (named_imports ((((import_specifier [(identifier) (string)]+ @named_import ("as" ([(identifier) (string)] @alias))?)) @import_with_alias) ","?)*)?))
                        (string) @source
                        ) @import"#).unwrap();
```

Let's take a look at how the above works. I want to capture an entire `import_statement` (which starts with `import`) as `@import`. Within each import statement, I want to capture the identifiers, whether it is a `@named_import` (`import { a } from b`) or `@namespace_import` (`import * as a from b`). I also want to capture aliases, but I also need to know which exported names are captured as aliases, which is why I wrapped the entire part up with `@import_with_alias`. Finally, I have to capture the package name, which is represented as a `string` that I captured as `@source`.

To show that it works, we have the following input:

```
"use strict";

import defaultExport from "module-name";
import * as name from "module-name";
import { export1, export2 as alias2, /* … */ } from "module-name";
import { "string name" as alias } from "module-name";
import defaultExport, { export1, /* … */ } from "module-name";
import defaultExport, * as name from "module-name";
```

This is the generated debug output:

```
[Line: 2, Col: 0, Match: 0, Name: import] Found: `import defaultExport from "module-name";`
[Line: 2, Col: 7, Match: 0, Name: named_import] Found: `defaultExport`
[Line: 2, Col: 26, Match: 0, Name: source] Found: `"module-name"`
[Line: 3, Col: 0, Match: 1, Name: import] Found: `import * as name from "module-name";`
[Line: 3, Col: 12, Match: 1, Name: namespace_import] Found: `name`
[Line: 3, Col: 22, Match: 1, Name: source] Found: `"module-name"`
[Line: 4, Col: 0, Match: 2, Name: import] Found: `import { export1, export2 as alias2, /* … */ } from "module-name";`
[Line: 4, Col: 9, Match: 2, Name: import_with_alias] Found: `export1`
[Line: 4, Col: 9, Match: 2, Name: named_import] Found: `export1`
[Line: 4, Col: 18, Match: 2, Name: import_with_alias] Found: `export2 as alias2`
[Line: 4, Col: 18, Match: 2, Name: named_import] Found: `export2`
[Line: 4, Col: 29, Match: 2, Name: alias] Found: `alias2`
[Line: 4, Col: 54, Match: 2, Name: source] Found: `"module-name"`
[Line: 5, Col: 0, Match: 3, Name: import] Found: `import { "string name" as alias } from "module-name";`
[Line: 5, Col: 9, Match: 3, Name: import_with_alias] Found: `"string name" as alias`
[Line: 5, Col: 9, Match: 3, Name: named_import] Found: `"string name"`
[Line: 5, Col: 26, Match: 3, Name: alias] Found: `alias`
[Line: 5, Col: 39, Match: 3, Name: source] Found: `"module-name"`
[Line: 6, Col: 0, Match: 4, Name: import] Found: `import defaultExport, { export1, /* … */ } from "module-name";`
[Line: 6, Col: 7, Match: 4, Name: named_import] Found: `defaultExport`
[Line: 6, Col: 24, Match: 4, Name: import_with_alias] Found: `export1`
[Line: 6, Col: 24, Match: 4, Name: named_import] Found: `export1`
[Line: 6, Col: 50, Match: 4, Name: source] Found: `"module-name"`
[Line: 7, Col: 0, Match: 5, Name: import] Found: `import defaultExport, * as name from "module-name";`
[Line: 7, Col: 7, Match: 5, Name: named_import] Found: `defaultExport`
[Line: 7, Col: 27, Match: 5, Name: namespace_import] Found: `name`
[Line: 7, Col: 37, Match: 5, Name: source] Found: `"module-name"`
```

This may not be foolproof, and there may still be cases in which the query fails to match. I will be writing unit tests for this to ensure that I am able to parse all that I need correctly. For now, this is a first-cut attempt at parsing TypeScript and to get myself used to `tree-sitter`'s DSL. There will be much more parsing and semantic analysis needed before I am anywhere close to being able to perform reachability analysis.