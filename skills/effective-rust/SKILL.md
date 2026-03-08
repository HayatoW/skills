---
name: effective-rust
description: Rust の型設計と Option/Result の扱いを改善するための指針を与える Skill。Rust コードの設計・実装・レビュー時に、bool 引数の置き換え、newtype と enum の使い分け、ADT による不変条件の表現、Option による欠損値表現、および Option/Result では match ではなく変換メソッド・if let・? の利用を検討するときに使う。
---

# Effective Rust

Rust のデータ構造と API を型で安全に表現し、Option と Result を簡潔に扱うための指針。

## When to use

- Rust の API 設計を改善したいとき
- `bool` 引数や `bool` フィールドが意味を隠しているとき
- 構造体の組み合わせで表現された不変条件を型に押し込みたいとき
- `String` の未設定値を空文字で表していそうなとき
- `Option` / `Result` を `match` で冗長に書いているとき、またはエラー伝搬を簡潔にしたいとき
- Rust コードレビューで、型安全性・保守性・可読性の観点から改善案を出したいとき

## Instructions

1. まず、値の意味が型に表れているか確認する。特に `bool`、空文字、相互依存する複数フィールドを優先的に見る。
2. `bool` 引数が登場したら、呼び出し側の可読性を確認する。`true` / `false` だけでは意味が読めないなら、意味を持つ型へ置き換える。
3. 引数が本質的に二値で、将来も値の種類が増えないなら `newtype` を検討する。将来ほかの値を取りうるなら `enum` を優先する。
4. 複数フィールドの組み合わせに不変条件があるなら、フィールド付き `enum` を使って代数データ型として表現し、無効な状態を作れない設計を優先する。
5. 「値が存在しない」状態を表す文字列や数値の番兵値を見つけたら、`Option<T>` に置き換えられないか検討する。
6. `Option` / `Result` に対して、値だけ欲しいときは `if let Some(x) = ...`、失敗時は panic でよいときは `unwrap()`、エラーを伝搬するときは `?` を検討し、定型的な `match` を減らす。
7. 提案時は「型安全性」「可読性」「保守性」「不変条件をコンパイル時に表現できるか」を短く説明する。

## Design rules

### `bool` 引数は意味のある型に置き換える

`bool` は呼び出し側で意味が失われやすい。引数の意図が値だけで伝わらない場合、`enum` または `newtype` を使う。

```rust
// 改善前
print_page(/* both_sides = */ true, /* color = */ false);
```

```rust
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

print_page(Sides::Both, Output::BlackAndWhite);
```

### `newtype` と `enum` を使い分ける

- 値が常に二値であることが本質なら `newtype` を検討する
- 将来、値の種類が増える可能性があるなら `enum` を使う

判断に迷う場合は、拡張余地を残せる `enum` を優先する。

### 不変条件は ADT で表現する

構造体と `bool` の組み合わせでルールを表すのではなく、無効な状態を表現できない型へ寄せる。

```rust
// 望ましくない例
struct DisplayProps {
    x: u32,
    y: u32,
    monochrome: bool,
    // monochrome が真なら fg_color は (0, 0, 0) でなければならない
    fg_color: RgbColor,
}
```

```rust
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

### 欠損値は `Option<T>` で表す

存在しない可能性のある `String` は `Option<String>` で表現する。`""` は「未設定」なのか「空文字」なのかが曖昧になるため避ける。

### Option と Result では match ではなく変換を使う

`match` より標準ライブラリの変換メソッド・`if let`・`?` を使うと、コンパクトで意図が明確になる。

**値だけ欲しく、None/Err のときは無視してよい場合**

```rust
// 改善前
match &s.field {
    Some(i) => println!("field is {i}"),
    None => {}
}

// 改善後
if let Some(i) = &s.field {
    println!("field is {i}");
}
```

**失敗時は panic でよく、以降は成功を前提に書く場合**

`unwrap()` は panic するので、panic を選ぶのと同じ。本当に回復不能なエラーである場合に使う。

```rust
let f = std::fs::File::open("/etc/passwd").unwrap();
```

失敗が「エラー」というより単なる欠損（例: 空スライスの `first()`）なら、返り値は `Option` にすべき。エラーに有用な情報があるなら `Result` を使う。`Result` には `#[must_use]` があり、無視するとコンパイラが警告する。

**エラーを伝搬する場合**

`?` は Err にマッチして変換し、`return Err(...)` を生成するシンタックスシュガー。手書きの `match` より簡潔になる。

```rust
// 改善前
let f = match std::fs::File::open("/etc/passwd") {
    Ok(f) => f,
    Err(e) => return Err(From::from(e)),
};

// 改善後
let f = std::fs::File::open("/etc/passwd")?;
```

これらは `#[inline]` 付きのジェネリック関数なので、手で書いた場合と同等のコードにコンパイルされる。

## Review checklist

- `bool` が API の意味を隠していないか
- `enum` にすべき状態分岐を、複数フィールドで無理に表していないか
- 型で表現できる不変条件を、コメントや実行時チェックに頼っていないか
- 欠損値を番兵値で表していないか
- `Option` / `Result` を定型的な `match` で書いていないか（`if let`・`unwrap`・`?` で簡潔にできるか）
- 提案が呼び出し側の読みやすさを改善しているか
