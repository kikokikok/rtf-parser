# rtf-parser
[![Crates.io](https://img.shields.io/crates/v/rtf-parser.svg?style=flat-square&color=orange)](https://crates.io/crates/rtf-parser)
![Crates.io License](https://img.shields.io/crates/l/rtf-parser?style=flat-square)
[![Crates.io Total Downloads](https://img.shields.io/crates/d/rtf-parser?label=Crates.io%20Downloads&style=flat-square&color=violet)](https://crates.io/crates/rtf-parser)
[![NPM Total Downloads](https://img.shields.io/npm/d18m/rtf-parser-wasm?label=NPM%20Downloads&style=flat-square&color=red)](https://www.npmjs.com/package/rtf-parser-wasm)
[![docs.rs](https://img.shields.io/docsrs/rtf-parser?style=flat-square)](https://docs.rs/rtf-parser)

A safe Rust RTF parser &amp; lexer library designed for speed and memory efficiency, with no external dependencies. 

It implements the last version of the RTF specification (1.9), with modern UTF-16 unicode support. 

The official documentation is available at [docs.rs/rtf-parser](https://docs.rs/rtf-parser).

## Installation
This library can be installed using cargo with the CLI :  
```bash
 cargo add rtf-parser
 ```
Or add `rtf-parser = "<last-version>"` under **[dependencies]** in your `Cargo.toml`.

If you want to use the WASM version in JavaScript, you can add this module via NPM : 
```node
npm i rtf-parser-wasm
```
Or add `"rtf-parser-wasm": "<last-version>"` in the  **dependencies** in your `package.json`.

## Design
The library is split into 2 main components:
1. The lexer
2. The parser

The lexer scans the document and returns a `Vec<Token>` which represent the RTF file in a code-understandable manner.
These tokens can then be passed to the parser to transcript it to a real document : `RtfDocument`.
```rust
use rtf_parser::{ Lexer, Token, Parser, RtfDocument };

fn main() -> Result<(), Box<dyn Error>> {
    let tokens: Vec<Token> = Lexer::scan("<rtf>")?;
    let parser = Parser::new(tokens);
    let doc: RtfDocument = parser.parse()?;    
}
```

or in a more concise way :

```rust 
use rtf_parser::RtfDocument;

fn main() -> Result<(), Box<dyn Error>> {
    let doc: RtfDocument = RtfDocument::try_from("<rtf>")?;    
}
```

The `RtfDocument` struct implement the `TryFrom` trait for : 
- `&str`
- `String`
- `&mut std::fs::File`  

and a `from_filepath` constructor that handle the i/o internally. 

The error returned can be a `LexerError` or a `ParserError` depending on the phase wich failed.  


An `RtfDocument` is composed with : 
- the **header**, containing among others the font table, the color table and the encoding.
- the **body**, which is a `Vec<StyledBlock>`

A `StyledBlock` contains all the information about the formatting of a specific block of text.  
It contains a `Painter` for the text style, a `Paragraph` for the layout, and the text (`String`).
The `Painter` is defined below, and the rendering implementation depends on the user.
```rust
pub struct Painter {
    pub font_ref: FontRef,
    pub font_size: u16,
    pub bold: bool,
    pub italic: bool,
    pub underline: bool,
    pub superscript: bool,
    pub subscript: bool,
    pub smallcaps: bool,
    pub strike: bool,
}
```

The layout information are exposed in the `paragraph` property :
```rust
pub struct Paragraph {
    pub alignment: Alignment,
    pub spacing: Spacing,
    pub indent: Indentation,
    pub tab_width: i32,
}
```
It defined the way a block is aligned, what spacing it uses, etc...

You also can extract the text without any formatting information, with the `to_text()` method of the `RtfDocument` struct.

```rust
fn main() -> Result<(), Box<dyn Error>> {
    let rtf = r#"{\rtf1\ansi{\fonttbl\f0\fswiss Helvetica;}\f0\pard Voici du texte en {\b gras}.\par}"#;
    let tokens = Lexer::scan(rtf)?;
    let document = Parser::new(tokens)?;
    let text = document.to_text();
    assert_eq!(text, "Voici du texte en gras.");
}
```

## Examples 
A complete example of rtf parsing is presented below : 
```rust
use rtf_parser::Lexer;
use rtf_parser::Parser;

fn main() -> Result<(), Box<dyn Error>> {
    let rtf_text = r#"{ \rtf1\ansi{\fonttbl\f0\fswiss Helvetica;}\f0\pard Voici du texte en {\b gras}.\par }"#;
    let tokens = Lexer::scan(rtf_text)?;
    let doc = Parser::new(tokens).parse()?;
    assert_eq!(
        doc.header,
        RtfHeader {
            character_set: Ansi,
            color_table: ColorTable::Default(),
            font_table: FontTable::from([
                (0, Font { name: "Helvetica", character_set: 0, font_family: Swiss })
            ])
        }
    );
    assert_eq!(
        doc.body,
        [
            StyleBlock {
                painter: Painter { font_ref: 0, font_size: 0, bold: false, italic: false, underline: false },
                paragraph: Paragraph {
                    alignment: LeftAligned,
                    spacing: Spacing { before: 0, after: 0, between_line: Auto, line_multiplier: 0, },
                    indent: Indentation { left: 0, right: 0, first_line: 0, },
                    tab_width: 0,
                },
                text: "Voici du texte en ",
            },
            StyleBlock {
                painter: Painter { font_ref: 0, font_size: 0, bold: true, italic: false, underline: false },
                paragraph: Paragraph {
                    alignment: LeftAligned,
                    spacing: Spacing { before: 0, after: 0, between_line: Auto, line_multiplier: 0, },
                    indent: Indentation { left: 0, right: 0, first_line: 0, },
                    tab_width: 0,
                },
                text: "gras",
            },
            StyleBlock {
                painter: Painter { font_ref: 0, font_size: 0, bold: false, italic: false, underline: false },
                paragraph: Paragraph {
                    alignment: LeftAligned,
                    spacing: Spacing { before: 0, after: 0, between_line: Auto, line_multiplier: 0, },
                    indent: Indentation { left: 0, right: 0, first_line: 0, },
                    tab_width: 0,
                },
                text: ".",
            },
        ]
    );
    return Ok(());
}
```

# WASM
This crate also compiles to WASM, and exposes the function `parse_rtf` to JS & TS, with proper type declarations.
The TS API is the same as the Rust one, except for the `Lexer` & the `Parser`. Due to performance reasons, those can't be exposed directly in JS and are internally used in WASM. 

## With NPM 
To use this module with NPM, you have to import it and initialize it :
```ts
import init, { parse_rtf } from 'rtf-parser-wasm'
init().then(() => {
    let document = parse_rtf("<rtf>")
})
```

## Without NPM
You have to downlod the `pkg/` folder, and then import the `rtf_parser.js` script.
```ts 
import init, { parse_rtf } from '../pkg/rtf_parser.js'
```
A complete example is provided in `examples/wasm/`.

### Vite
If you are using Vite, don't forget to add this snippet to your `vite.config.js`, for the WASM to be served correctly :
```ts 
import { defineConfig } from 'vite'

export default defineConfig({
    optimizeDeps: {
        exclude: ["rtf-parser-wasm"]
    }
})
``` 

## Known limitations
For now, the `\bin` keyword is not taken into account. As its content is text in binary format, it can mess with the lexing algorithm, and crash the program. 
Future support for the binary will soon come.

The base64 images are not supported as well, but can safely be parsed. 

## Benchmark
For now, there is no comparable crates to [`rtf-parser`](https://crates.io/crates/rtf-parser).  
However, the `rtf-grimoire` crate provide a similar *Lexer*. Here is a quick benchmark of the lexing and parsing of a [500kB rtf document](./resources/tests/file-sample_500kB.rtf).

| Crate                                                                 | Version | Duration |
|-----------------------------------------------------------------------|:-------:|---------:|
| [`rtf-parser`](https://crates.io/crates/rtf-parser)                   | v0.3.0  |   _7 ms_ |
| [`rtf-grimoire`](https://crates.io/crates/rtf-grimoire) (only lexing) | v0.2.1  |  _13 ms_ |

*This benchmark has been run on an Intel MacBook Pro, with the release build*.  



