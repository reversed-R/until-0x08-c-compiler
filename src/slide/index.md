---
marp: true
theme: "gaia"
header: "Cコンパイラ書いています"
footer: "ゃー@reversed_R UNTIL.LT 0x08 2025/07/19"
backgroundColor: white
<!-- headingDivider: 1 # divide pages by header 1 (# header) -->
paginate: true # display page number below
size: 16:9
math: katex
---

<style scoped>
  .title {
    text-align: center;
    font-size: 2rem;
  }

  .sub {
    text-align: center;
    font-size: 1.2rem;
  }

  .where {
    text-align: center;
    font-size: 1rem;
  }
</style>

<h1 class="title">セキュキャンCコンパイラゼミ行くので、Cコンパイラ書いています</h1>
<h2 class="sub">〜いわゆる事前学習ってやつ〜</h2>
<h2 class="where">UNTIL.LT 0x08 2025/07/19</h2>

---

# とりあえず、自己紹介

<img src="./images/icon.jpg" width="15%">

こんにちは。
情報科学類2年のゃー(reversed_R)です。

Twitter: [@reversed_R](https://x.com/reversed_R)
GitHub: [reversed-R](https://github.com/reversed-R)
HP: [https://reversed-r.dev](https://reversed-r.dev)

---

# セキュキャン全国Cコンパイラゼミに行くことに

---

## セキュキャン とは

<img src="https://upload.wikimedia.org/wikipedia/commons/b/b2/IPA_logo.png" width="10%"> [(独立行政法人情報処理推進機構)](https://www.ipa.go.jp/)が主催する、全国からオタクを募り、夏の暑い中東京の施設に5日間軟禁する大会。

- 夏にやるやつを`全国`大会といい
  - 中高生向けの`ジュニア`
  - 全国卒業生向けの`ネクスト`も合わせて開催
- 適当な時期に各地で`ミニ`キャンプもやっている

<img class="doppousan" src="https://pbs.twimg.com/media/ElUhDtrVcAAQpXe?format=jpg&name=900x900" width="15%">

<style scoped>
  .doppousan {
    position: fixed;
    right: 2%;
    bottom: 2%;
    width: 30%;
  }
</style>

---

# セキュキャン全国Cコンパイラゼミに行くことに

全国大会の様々なゼミがあるうちの**Cコンパイラ**自作ゼミに通りました。

講師: [hsjoihs](https://x.com/hsjoihs)さん <img src="https://pbs.twimg.com/profile_images/876768132525719552/ZCUSslij_400x400.jpg" width="10%">

---

# そういうわけで、Cコンパイラを書きます

リポジトリ: Rustで書いてます
[https://github.com/reversed-R/ya-cc](https://github.com/reversed-R/ya-cc)

補足:
セキュキャンCコンパイラゼミ初代講師の人(Rui Ueyama)がセキュキャン2019?での記憶をもとに書いたWeb資料(通称compilerbook)が参考になる。
[https://www.sigbus.info/compilerbook](https://www.sigbus.info/compilerbook)

---

# Cコンパイラの手順のおさらい

---

これを

```c
#include <stdio.h>

int main(int argc, char**argv) {
    printf("Hello, World!\n");
    return 0;
}
```

---

こうすればいいわけ

```asm
.LC0:
        .string "Hello, World!"
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-4], edi
        mov     QWORD PTR [rbp-16], rsi
        mov     edi, OFFSET FLAT:.LC0
        call    puts
        mov     eax, 0
        leave
        ret
```

---

## 1. プリプロセス

- マクロなどをしばく。ヘッダを`#include`に展開したり、`#define`に従って書き換えたり。

## 2. **真のコンパイル**

- アセンブリファイルができる

---

## 3. アセンブル

- オブジェクトファイルができる

## 4. リンク

- 関数などのアドレス解決など

\
このうち、**真のコンパイル**を主に考えます

---

# 真のコンパイルの手順

## 1. トークナイズする

- ソースファイルとして渡されたテキストの列をトークンに分解

## 2. パースする

- トークン列を定められたパターンで再帰的にパースしASTを吐く

---

## 3. 型チェックなど

- パースは文法的な意味とか知らないのでチェック

## 4. コード生成する

- 原始的には各シンボルごとにコードを吐けば良い

---

# 1. トークナイズする

- 文法上意味を持つトークン
  - 一部記号: `;`, `+`, `*`, `{`, ...
  - 予約語: `int`, `if`, `return`, ...
- その他有象無象の文字列
  - リテラル(即値): `328`, `0xA8`, `2.163`, `"Hello, World!\n"`, ...
  - 識別子: `a`, `num`, `char_ptr`, ...

大体、先頭から舐めていけばなんとかなる

---

諸説ですが、Rustで書くならこんな感じになるでしょう

```rs
#[derive(Clone, Debug, PartialEq)]
pub enum Token {
  String(String),  // string of remain characters
  IntLiteral(i64), // int literal
  If,              // if
  Else,            // else
  While,           // while
  Return,          // return
  Int,             // int (reserved word of type)
  LPare,           // (
  RPare,           // )
  LBrace,          // {
  RBrace,          // }
  LBracket,        // [
  RBracket,        // ]
  Plus,            // +
  Minus,           // -
  Asterisk,        // *
  Slash,           // /
  ...
}
```

---

# 2. パースする

トークンが得られました。これを**終端記号**と言います

とある文法要素(これを **非終端記号** と言います)は、これら終端記号の定められた連続で表せるはず

ということは、非終端記号である左辺は、終端または非終端記号の連続である右辺で表せるという形式の
BNFで展開可能な記号列のパターンを列挙できます

---

簡単のため四則演算とif文と関数があるCっぽい言語を考えます

```
program = fndec*
fndec = identifier "(" (identifier ",")* ")" "{" stmt* "}"
stmt = expr ";" | if "(" expr ")" stmt
expr = add
add = mul (( "+" | "-" ) mul)*
mul = primary (( "*" | "/" ) primary)*
primary = literal | identifier | "(" expr ")"
```

addとmulを分けているのは演算子の結合の優先度を表すため

---

簡単のため四則演算とif文と関数があるCっぽい言語を考えます

**program** = fndec*
**fndec** = `identifier` `(` (`identifier` (`,` `identifier`)\*)\* `)` `{` stmt* `}`
**stmt** = expr `;` | `{` stmt* `}` | `if` `(` expr `)` stmt (`else` stmt )?
**expr** = add
**add** = mul (( `+` | `-` ) mul)*
**mul** = primary (( `*` | `/` ) primary)\*
**primary** = `literal` | `identifier` | `(` expr `)` \
 | `identifier` `(` (`identifier` (`,` `identifier`)\*)\* `)`

(みなさんが分かりやすいかと思って`終端記号`だけ括ったバージョン)

---

パーサは、この各非終端記号のパース関数を書き、自身に他の非終端記号が含まれていればそれを再帰的に呼び出せばよい。

後は解釈して得られた必要な値を抽象構文木を表す適当なデータ型に詰めてやればよいです

---

```rs
impl Program {
  fn consume(
        tokens: &mut std::iter::Peekable<std::slice::Iter<'_, Token>>,
  ) -> Result<Self, ParseError> {
    let mut prog = Self { fns: vec![] };

    while let Ok(fn_dec) = FnDec::consume(tokens) {
        prog.fns.push(fn_dec);
    }

    Ok(prog)
  }
}
```

---

```rs
impl FnDec {
  fn consume(
      tokens: &mut std::iter::Peekable<std::slice::Iter<'_, Token>>,
  ) -> Result<Self, ParseError> {
    if let Some(Token::String(name)) = tokens.next() {
      if let Some(Token::LPare) = tokens.peek() {
        if let Ok(args) = ArgsDec::consume(tokens) {
          if let Ok(block) = BlockStmt::consume(tokens) {
            Ok(Self {
                name: name.clone(),
                args: args.args,
                stmts: block.stmts,
            })
          } else {
              Err(ParseError::InvalidToken)
          }
        } else {
            Err(ParseError::InvalidToken)
        }
      } else {
          Err(ParseError::InvalidToken)
      }
    } else {
        Err(ParseError::InvalidToken)
    }
  }
}
```

---

# 3. 型チェックなど

型チェックはまだやれてません
現代的な言語は型推論とかしまくった上で型の整合性をチェックしているわけです。偉すぎる

---

型以外にもチェックすべきことはあって、
例えばこれまで話したBNFで表現可能な言語表現は記号の列の並びを規定しているに過ぎないので、

```
a + 2 = 5;
```

みたいな不正な左辺値(代入不能なシンボル)に対する代入文を許してしまいます。
こういうチェックをこの段階で行っても良いと思います(コード生成の瞬間にチェックするでも良い)

---

# 4. コード生成

ついにコード生成に来ました

ここでプロセスのメモリ領域の使い方をおさらい
OSによるがx86_64でのLinuxについて考えると...

---

|                                       |
| :-----------------------------------: |
|              スタック ↓               |
|                  ...                  |
|                  ...                  |
|                  ...                  |
|               ヒープ ↑                |
|  BSS\* 領域 (未初期化グローバル変数)  |
| データ領域 (初期化済みグローバル変数) |
|       テキスト領域 (実行コード)       |

\*: _Block Started by Symbol_

---

一旦ヒープのことは忘れましょう
Cのような基礎的な言語はとりあえずスタック上でどうにかやりくりするように作られているはず
(実際、alloc系を呼び出さないとヒープにメモリ確保しない)

スタックは、値の`push`で伸び、`pop`で縮んだり、
関数のコール毎に伸び、リターンで縮んだり、を繰り返します

スタックの先端をスタックトップと言い、そのアドレスをスタックポインタなど呼びます
x86_64では`rsp`レジスタに格納されており、`push`や`pop`は単位byte分(8byte)`rsp`を加減算しつつ値を移動させているだけ

---

ところでCPUにはレジスタが載っている(x86_64なら16本, RISC-V64なら32本)

メモリの読み書きはレジスタに比べて遅いため、できるもんならレジスタ上で演算したい

が、難しい

初歩的な方法では難しいという話と、本数が限られているため無理という話

---

例えば

```c
int a = 3 + 4 + 2 - 9 - 7 + ...;
```

なら、

```asm
mov rax, 3
add rax, 4
add rax, 2
sub rax, 9
sub rax, 7
...
```

とやれば1本のレジスタのみで演算できる

---

しかし、

```c
int a = 3 * 4 + 2 / (9 - f(11) * 7) + ...;
```

なら、演算の優先度や再帰性や関数呼び出しを考えるとレジスタには載りきらないので...

スタックに`push`する

---

初歩的な実装として

各記号のコード生成で

1. 必要なら前の式の結果をスタックから`pop`し、
1. 演算結果をスタックに`push`する

としておき、ASTにしたがって各記号の生成関数を呼び出す
とすれば一応は成立します
とりあえず最適化は済んでいないが動くコードを吐き、次に最適化を入れれば良いでしょう

---

```rs
impl  MulExpr {
  fn generate(&self, vars: &mut crate::generator::x86_64::globals::Vars) {
    self.left.generate(vars);

    for mul in &self.rights {
      match mul.op {
        MulOperator::Mul => {
          mul.right.generate(vars);

          println!("pop rdi");
          println!("pop rax");
          println!("imul rax, rdi");
          println!("push rax");
        }
        MulOperator::Div => {
          mul.right.generate(vars);

          println!("pop rdi");
          println!("pop rax");
          println!("cqo");
          println!("idiv rdi");
          println!("push rax");
        }
      }
    }
  }
}
```

---

## 関数をコールしたい!

関数、コールしたいですねぇ

関数のコールは実行しているコードの位置がぶっ飛ぶので、元に戻るために`call`の次のインストラクションのアドレス(**リターンアドレス**)を保持しておく必要があります

また、関数内でのローカル変数をスタックに載せたいのでそのことも考えてやる必要があります
ローカル変数はコンパイラがサイズと数を知っているので、スタックの基準位置(ベースポインタ)からのオフセットでアクセスすれば良さそう

---

さらに先程のような計算途中の値もスタックに載りうるので、

1. 関数をコールするときにリターンアドレスをスタックに格納
1. 関数が呼び出された時点でその関数内でのベースポインタを決める(当然呼び出した側の関数も同じことをしておりその値はレジスタに入っているため、それが失われないようにスタックに格納してやる必要がある)
1. ローカル変数が必要とする分だけスタックトップを下げてやる
1. 適当な演算の途中の値はそれ以降に入る

---

fがgを呼び出すとき、

|                           |
| :-----------------------: |
|            ...            |
|        ... <- rbp         |
|     fでのローカル変数     |
| fでのローカル変数2 <- rsp |

---

|                         |
| :---------------------: |
|           ...           |
|       ... <- rbp        |
|    fでのローカル変数    |
|   fでのローカル変数2    |
| リターンアドレス <- rsp |

---

|                              |
| :--------------------------: |
|             ...              |
|             ...              |
|      fでのローカル変数       |
|      fでのローカル変数2      |
|       リターンアドレス       |
| コール時点でのrbp <- rsp,rbp |

---

|                           |
| :-----------------------: |
|            ...            |
|            ...            |
|     fでのローカル変数     |
|    fでのローカル変数2     |
|     リターンアドレス      |
| コール時点でのrbp <- rbp  |
|     gでのローカル変数     |
| gでのローカル変数2 <- rsp |

---

逆に関数からリターンするときは

1. 戻り値はx86_64なら`rax`レジスタに入っているので気にしなくて良い
1. スタックポインタをベースポインタと同じにすることで、ローカルのすべての値を忘れる
1. ベースポインタを`pop`して呼び出し元でのベースポインタを取り戻す
1. リターンアドレスを取り出して戻る(`ret`がやってくれる)

---

fがgを呼び出したので、fに戻るとき、

|                           |
| :-----------------------: |
|            ...            |
|            ...            |
|     fでのローカル変数     |
|    fでのローカル変数2     |
|     リターンアドレス      |
| コール時点でのrbp <- rbp  |
|     gでのローカル変数     |
| gでのローカル変数2 <- rsp |

---

|                              |
| :--------------------------: |
|             ...              |
|             ...              |
|      fでのローカル変数       |
|      fでのローカル変数2      |
|       リターンアドレス       |
| コール時点でのrbp <- rsp,rbp |

---

|                         |
| :---------------------: |
|           ...           |
|       ... <- rbp        |
|    fでのローカル変数    |
|   fでのローカル変数2    |
| リターンアドレス <- rsp |

---

|                           |
| :-----------------------: |
|            ...            |
|        ... <- rbp         |
|     fでのローカル変数     |
| fでのローカル変数2 <- rsp |

やった〜、元に戻った〜

---

つまり

- 関数の冒頭(プロローグ)

```asm
push rbp
mov rbp, rsp
sub rsp, 8?*ローカル変数の数
```

- 関数の末尾(エピローグ)

```asm
mov rsp, rbp
pop rbp
ret
```

とすれば良い

---

### その他、関数の引数の渡し方など

最初の6個程度までは特定のレジスタで渡すというABIが規定されていたり、スタックにプッシュして渡すという規定がされていたり

x86_64(System V AMD64 ABI)では、最初の6個は`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`で渡して残りはスタック

浮動小数点数はまた別のレジスタがある

---

まあ、そんなこんなで現在実装しており、現在、
こんな感じのCもどきをパースでき、

```
main(){
  a = 0;
  return sub(subsub(6), 1);
}

sub(a, b){
  return 4;
  3;
}

subsub(a){
  return 0;
}
```

---

```asm
.intel_syntax noprefix
.globl main
main:
push rbp
mov rbp, rsp
and rsp, -16
sub rsp, 4
push 0
pop rdi
mov [rbp - 4], rdi
push rdi
pop rax
push 6
pop rcx
call subsub
push rax
pop rcx
push 1
pop rdx
call sub
push rax
pop rax
leave
ret
leave
ret
```

---

```asm
sub:
push rbp
and rsp, -16
sub rsp, 8
mov [rbp - 4], rcx
mov [rbp - 8], rdx
push 4
pop rax
leave
ret
push 3
pop rax
leave
ret
subsub:
push rbp
and rsp, -16
sub rsp, 4
mov [rbp - 4], rcx
push 0
pop rax
leave
ret
leave
ret
```

---

という、それっぽいアセンブリを吐き、動きそうだがちょっと戻り値が変だったりセグフォしたりみたいな感じになっていたりします

制御構文もさっさと作ろうと思っていたのですが間に合わず、最適化していく必要もあり、...

今後の健闘に期待、という感じです

---

# おわり

全人類コンパイラを書いているようなので(?)、ぜひ書くと良いと思います

ありがとうございました。
