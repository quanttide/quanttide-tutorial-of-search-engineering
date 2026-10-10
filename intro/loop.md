# 从零写一个搜索程序：让搜索变准的最小循环

读完你会从**空目录**做出一个能搜中文的程序，然后把它调准，并用数字证明它变准了。四段路：建项目 → 让它认中文 → 让它搜真文件 → 调一圈。

不 clone 任何项目，代码一步步写。所有输出为 2026-10-10 本机实测（Rust 1.93.1、tantivy 0.26.2）。

## 准备：装 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
cargo --version   # 有版本号就成
```

## 第 0 步：建项目

```bash
cargo new search-demo
cd search-demo
cargo run
```

```
Hello, world!
```

`cargo new` 生成一个能编译的空项目，`cargo run` 编译并执行。看到这句，工具链就通了。

## 第 1 步：写最小版——两篇假文档

**先加依赖**。打开 `Cargo.toml`，在 `[dependencies]` 下加一行：

```toml
[dependencies]
tantivy = "0.26"
```

**再写 `src/main.rs`**（先看懂三段：定字段 → 写文档 → 查询）：

```rust
use tantivy::collector::TopDocs;
use tantivy::query::QueryParser;
use tantivy::{doc, schema::{Field, Schema, Value, STORED, STRING, TEXT}};
use tantivy::{Index, IndexReader};

fn main() {
    // 1) 定字段：path 存起来好显示，body 是被搜的正文
    let mut sb = Schema::builder();
    let path_f: Field = sb.add_text_field("path", STRING | STORED);
    let body_f: Field = sb.add_text_field("body", TEXT);
    let schema = sb.build();

    // 2) 内存索引，写两篇假文档，提交
    let index = Index::create_in_ram(schema);
    let mut writer = index.writer(50_000_000).unwrap();
    writer.add_document(doc!(path_f => "a.md", body_f => "本站点索引的新鲜度很重要")).unwrap();
    writer.add_document(doc!(path_f => "b.md", body_f => "完全无关的另一篇")).unwrap();
    writer.commit().unwrap();

    // 3) 查「新鲜度」
    let reader: IndexReader = index.reader().unwrap();
    let searcher = reader.searcher();
    let parser = QueryParser::for_index(&index, vec![body_f]);
    let query = parser.parse_query("新鲜度").unwrap();
    let top: Vec<(f32, tantivy::DocAddress)> =
        searcher.search(&query, &TopDocs::with_limit(10).order_by_score()).unwrap();

    println!("命中 {} 篇", top.len());
    for (score, addr) in top {
        let d: tantivy::TantivyDocument = searcher.doc(addr).unwrap();
        println!("  {score:.3} {}", d.get_first(path_f).unwrap().as_str().unwrap());
    }
}
```

```bash
cargo run
```

```
命中 0 篇
```

**0 篇。** a.md 明明写着「新鲜度」。原因：程序把整句话当成**一个词**存进索引，查询「新鲜度」是另一个词，对不上。中文没有空格，得先有人把它切成词。

## 第 2 步：让中文能被切开

**依赖加一行**（`Cargo.toml`）：

```toml
[dependencies]
tantivy = "0.26"
tantivy-jieba = "0.21"
```

**`src/main.rs` 改三处**：

```rust
// ① import 里把 TEXT 换成下面三个
use tantivy::{doc, schema::{Field, Schema, TextFieldIndexing, IndexRecordOption, TextOptions, Value, STORED, STRING}};

// ② body 字段声明用 jieba 切词（替换原来的 TEXT 那行）
let indexing = TextFieldIndexing::default()
    .set_tokenizer("jieba")
    .set_index_option(IndexRecordOption::WithFreqsAndPositions);
let body_f: Field = sb.add_text_field("body", TextOptions::default().set_indexing_options(indexing));

// ③ 建索引之后、写入之前，注册这个切词器
index.tokenizers().register("jieba", tantivy_jieba::JiebaTokenizer::default());
```

```bash
cargo run
```

```
命中 1 篇
  1.862 a.md
```

**命中了。** jieba 把句子切成「本站点 / 索引 / 的 / 新鲜度 / 很 / 重要」，查询词也是切过的词，词对词就对上了。`1.862` 是 BM25 打的分——这个词在这篇里出现得多、且不是满篇都有，分就高，分高排前面。

## 第 3 步：换成真文件

假文档换成本地真实文件。在 `search-demo` **旁边**建一个语料目录：

```bash
mkdir -p ../mydocs
```

三篇内容（考题都藏在里面）：

```markdown
<!-- ../mydocs/a.md -->
# 索引维护笔记
索引要每天更新。索引的新鲜度决定搜索质量，索引过期就搜不到新东西。
本篇只讨论索引，不讨论别的。

<!-- ../mydocs/b.md -->
# 工具笔记
tantivy 是一个 Rust 写的搜索引擎库。

<!-- ../mydocs/c.md -->
# 购物清单
牛奶、面包、鸡蛋。
```

**`src/main.rs` 换成完整版**（新增遍历目录、存到磁盘、查询从命令行来）：

```rust
use std::fs;
use std::path::{Path, PathBuf};
use tantivy::collector::TopDocs;
use tantivy::query::QueryParser;
use tantivy::{doc, schema::{Field, Schema, TextFieldIndexing, IndexRecordOption, TextOptions, Value, STORED, STRING}};
use tantivy::{Index, IndexReader};

fn walk(dir: &Path, out: &mut Vec<PathBuf>) {
    let Ok(rd) = fs::read_dir(dir) else { return };
    for e in rd.flatten() {
        let p = e.path();
        if p.is_dir() {
            walk(&p, out);
        } else if p.extension().is_some_and(|x| x == "md") {
            out.push(p);
        }
    }
}

fn main() {
    let root = std::env::args().nth(1).expect("用法: cargo run -- <目录> <查询...>");
    let queries: Vec<String> = std::env::args().skip(2).collect();

    let mut files = Vec::new();
    walk(Path::new(&root), &mut files);

    let mut sb = Schema::builder();
    let path_f: Field = sb.add_text_field("path", STRING | STORED);
    let indexing = TextFieldIndexing::default()
        .set_tokenizer("jieba")
        .set_index_option(IndexRecordOption::WithFreqsAndPositions);
    let body_f: Field = sb.add_text_field("body", TextOptions::default().set_indexing_options(indexing));
    let schema = sb.build();

    fs::create_dir_all("/tmp/search-index").unwrap();
    let dir = tantivy::directory::MmapDirectory::open("/tmp/search-index").unwrap();
    let index = Index::open_or_create(dir, schema).unwrap();
    index.tokenizers().register("jieba", tantivy_jieba::JiebaTokenizer::default());

    let mut writer = index.writer(50_000_000).unwrap();
    let mut n = 0usize;
    for p in &files {
        let Ok(body) = fs::read_to_string(p) else { continue };
        let rel = p.strip_prefix(&root).unwrap_or(p).display().to_string();
        writer.add_document(doc!(path_f => rel, body_f => body)).unwrap();
        n += 1;
    }
    writer.commit().unwrap();
    eprintln!("已索引 {n} 个 Markdown 文件");

    let reader: IndexReader = index.reader().unwrap();
    let searcher = reader.searcher();
    let parser = QueryParser::for_index(&index, vec![body_f]);
    for q in &queries {
        println!("查询「{q}」");
        let query = parser.parse_query(q).unwrap();
        let top: Vec<(f32, tantivy::DocAddress)> =
            searcher.search(&query, &TopDocs::with_limit(10).order_by_score()).unwrap();
        if top.is_empty() { println!("  无结果"); }
        for (score, addr) in top {
            let d: tantivy::TantivyDocument = searcher.doc(addr).unwrap();
            println!("  {score:.3} {}", d.get_first(path_f).unwrap().as_str().unwrap());
        }
    }
}
```

> ⚠️ **每条命令跑之前先删索引**：`rm -rf /tmp/search-index`。程序每次运行都把文件重新加进索引，目录不删就重复计入——实测第二次跑之后同一文件出现两行、分数漂移，这时测出的「变准了」全是假数。

```bash
rm -rf /tmp/search-index && cargo run --release -- ../mydocs "索引 新鲜度"
```

```
已索引 3 个 Markdown 文件
查询「索引 新鲜度」
  3.108 a.md
  0.495 b.md
```

## 第 4 步：最小循环——单子 → 跑 → 改一处 → 再跑

改之前先记下「应该出什么」，这张单子叫**评测**。三条：

1. 查「检索」→ 应该出 a.md（a 写满「索引」，但一个「检索」都没有——正好当考题）
2. 查 `tantsivy`（我手打错的词）→ 应该出 b.md
3. 查「索引 新鲜度」和「新鲜度 索引」→ 应该一模一样

```bash
rm -rf /tmp/search-index && cargo run --release -- ../mydocs \
  "检索" "tantsivy" "tantsivy tantivy" "检索 索引" "索引 新鲜度" "新鲜度 索引"
```

```
查询「检索」
  无结果                                  ← 考题 1 不及格：它只认字面
查询「tantsivy」
  无结果                                  ← 考题 2 不及格：它不纠错
查询「tantsivy tantivy」
  1.034 b.md                              ← 改法 A：错词正词一起查，追回
查询「检索 索引」
  0.765 a.md                              ← 改法 B：近义词一起查，追回且排第一
  0.495 b.md
查询「索引 新鲜度」
  3.108 a.md
  0.495 b.md                              ← 考题 3 及格：换词序结果一样
查询「新鲜度 索引」
  3.108 a.md
  0.495 b.md
```

判卷：考题 3 直接过；考题 1、2 不及格，各用一条改写救回。**这就是循环的完整一圈**——单子 → 跑 → 改一处 → 再跑，进步是数字说的，不是感觉说的。

三条纪律：

- 单子不许改，改了就没有可比性。
- 一次只改一处，同时改两处不知道功劳是谁的。
- **换大语料要重新判卷**。同样那条改法 B，在 6226 篇的全仓上跑「检索 索引」，目标文件进不了前十——语料越大、「索引」越满篇，它把分抬得越平。小语料上灵的招，大语料上要拿单子重测。

## 收尾

你刚跑完第一圈。想继续调，就往单子里加查询，改字段、改改写、改打分，每轮只动一处。

- 原型能力边界与踩坑：[quanttide-search-lab](https://github.com/quanttide/quanttide-search-lab)
- 索引本身难在哪：[index-difficulties](https://github.com/quanttide/quanttide-search-lab/blob/main/docs/index-difficulties.md)
