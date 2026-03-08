# 型

## データ構造を表現するために型システムを用いる

### enum

例えば bool 型の引数を取る関数の可読性と保守性を高めることができる。

```rust
// 改善前
print_page(/* both_sides = */ true, /* color = */ false);
```

```rust
// 改善後
enum Sides {
    Both,
    Single,
}

enum Output {
    BlackAndWhite,
    Color,
}

fn print_page(sides: Sides, output: Output) {
    // ...
}

// このほうが型安全だし、可読性が高い
print_page(Sides::Both, Output::BlackAndWhite);
```

newtype パターンを用いて bool をラップしても、同じように安全性と保守性が得られる。
引数が常に bool であることがわかっているなら newtype パターンを使ったほうが良い。
将来他の値をとる可能性があるなら enum を使う。

### フィールド付き enum

enum を 代数データ型 (ADT: Algebraic Data Type) として使うことができる。
この機能を用いると、プログラムのデータ構造の不変条件を Rust の型システムにエンコードできる。

```rust
// 望ましくない振る舞い
struct DisplayProps {
    x: u32,
    y: u32,
    monochrome: bool,
    // `monochrome` が真なら `fg_color` は (0, 0, 0) でなければならない
    fg_color: RgbColor,
}
```

```rust
// enum で置き換えるのに最適
enum Color {
    Monochrome,
    Foreground(RgbColor),
}

struct DisplayProps {
    x: u32,
    y: u32,
    color: Color,
}
```

### 多用される enum 型

- 存在しない可能性のある String は Option<String> として表現する
  - `""` は存在しないのか空文字なのかが曖昧であるため

## 型システムを用いて共通の挙動を表現する

- 特になし

## Option と Result に対しては match を用いずに変換する

match ではなく、標準ライブラリが提供するさまざまな変換メソッドを使用すると、よりコンパクトで、定型的で、意図がはっきりとわかるコードになる。

値のみが重要で値がない場合は無視していい場合

```rust
// 改善前
struct S {
    field: Option<i32>,
}

let s = S { field: Some(42) };
match &s.field {
    Some(i) => println!("field is {i}"),
    None => {}
}
```

```rust
// 改善後
if let Some(i) = &s.field {
    println!("field is {i}");
}
```

失敗時に panic! を行うようにするとプログラムは即時に終了するが、残りのコードは成功したことを前提に書くことができる。これを明示的に match を用いて書くとコードが煩雑になる。

```rust
// 改善前
let result = std::fs::File::open("/etc/passwd");
let f = match result {
    Ok(f) => f,
    Err(_e) => panic!("Failed to open /etc/passwd!"),
}
// これ以降は、`f` が有効な `std::fs::File` であると想定
```

```rust
// 改善後
let f = std::fs::File::open("/etc/passwd").unwrap();
```

このヘルパ関数は panic! するので、これらを用いるのは **panic!** を選択するのと同じである。

上記の例のように、ファイルをオープンできなかったのは間違いなくエラーであるが、スライスが空の場合にスライスから最初の要素を取り出す first() メソッドが、失敗するのは本当のエラーではない。
このような場合は Option を返り値の型として用いることで表現したほうが良い。
Option と Result のいずれかを選択するには判断が必要だが、エラーに何か有用な情報があるのなら、Result を用いたほうが良い。

Result には #[must_use] 属性があり、コードが返された Result を無視した場合には、コンパイラが警告を発する。

```rust
// 明示的に match を用いればエラーを伝搬できるが、定型句をたくさん書かなければならない
fn find_user(username: &str) -> Result<UserId, std::io::Error> {
    let f = match std::fs::File::open("/etc/passwd") {
        Ok(f) => f,
        Err(e) => return Err(From::from(e)),
    };
    // ...
}
```

定型句を減らす鍵となるのが、Rust のクエスチョンマーク演算子?である。
これは、シンタックスシュガーで選択肢 Err へマッチングし、必要に応じてエラー型を変換し、return Err(...) 式を生成する。

```rust
// 改善後
fn find_user(username: &str) -> Result<UserId, std::io::Error> {
    let f = std::fs::File::open("/etc/passwd")?;
    // ...
}
```

一般にはメソッド呼び出しのコストもかからない。これらはすべて #[inline] とマークされたジェネリック関数なので、コンパイルした結果生成される機械語は手で書いた場合のコードと全く同じになる。
以上より、「明示的な match 式ではなく Option と Result の変換を用いたほうが良い」。
