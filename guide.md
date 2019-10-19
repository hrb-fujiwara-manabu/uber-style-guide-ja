# 導入
コーディングスタイルは私達のコードを統治する規則です。
これらのスタイルは、gofmt がやってくれることから少しだけ発展したものです。

このガイドのゴールはUber社内でのGoのコードでやるべき、もしくはやるべからずを説明し、コードの複雑さを管理することです。
これらのルールはコードを管理しやすくし、かつエンジニアがGoの言語機能をより生産的に利用できるようにします。

このガイドは元々同僚がGoを使ってより開発しやすくするために[Prashant Varanasi]( https://github.com/prashantv )と[Simon Newton]( https://github.com/nomis52 )によって作成されました。
長年にわたって多くのフィードバックを受けて修正されています。

このドキュメントはUber社内で使われる規則を文書化したものです。
多くは以下のリソースでも見ることができるような一般的なものです。

1. Effective Go
2. The Go common mistakes guide

全てのコードは `golint` や `go vet` を通してエラーが出ない状態にするべきです。
エディタに以下の設定を導入することを推奨しています。

1. 保存するごとに `goimports` を実行する
2. `golint` と `go vet` を実行してエラーがないかチェックする

Goのエディタのサポートについては以下の資料を参考にしてください。
https://github.com/golang/go/wiki/IDEsAndTextEditorPlugins

# ガイドライン
## Pointers to Interfaces
インタフェースをポインタとして渡す必要はほぼありません。
インタフェースは値として渡すべきです。
ただインタフェースを実装している要素はポインタでも大丈夫です。

インタフェースには2種類あります。

1. 型付けされた情報へのポインタ。これは type と考えることができます。
2. データポインタ。格納されたデータがポインタならそのまま使えます。格納されたデータが値ならその値のポインタになります。

もしインタフェースのメソッドがそのインタフェースを満たした型のデータをいじりたいなら、インタフェースの裏側の型はポインタである必要があります。

## Receivers and Interfaces
レシーバーが値のメソッドはレシーバーがポインタでも呼び出すことができますが、逆はできません。

```Go
type S struct {
  data string
}

func (s S) Read() string {
  return s.data
}

func (s *S) Write(str string) {
  s.data = str
}

sVals := map[int]S{1: {"A"}}

// You can only call Read using a value
sVals[1].Read()

// This will not compile:
//  sVals[1].Write("test")

sPtrs := map[int]*S{1: {"A"}}

// You can call both Read and Write using a pointer
sPtrs[1].Read()
sPtrs[1].Write("test")
```

同じように、メソッドのレシーバーが値型でも、ポインタがインタフェースを満たしているとみなされます。

```
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

s1Val := S1{}
s1Ptr := &S1{}
s2Val := S2{}
s2Ptr := &S2{}

var i F
i = s1Val
i = s1Ptr
i = s2Ptr

// The following doesn't compile, since s2Val is a value, and there is no value receiver for f.
//   i = s2Val
```

Effective Go の [Pointers vs Values]( https://golang.org/doc/effective_go.html#pointers_vs_values )を見るとよいでしょう。

## Zero-value Mutexes are Valid
`sync.Mutex` や `sync.RWMutex` はゼロ値でも有効です。ポインタで扱う必要はありません。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
mu := new(sync.Mutex)
mu.Lock()
```

</td><td>

```go
var mu sync.Mutex
mu.Lock()
```

</td></tr>
</tbody></table>

もし構造体のポインタを使う場合、mutexはポインタでないフィールドにする必要があります。
外部に公開されてない構造体なら、mutexを埋め込みで使うこともできます。


<table>
<tbody>
<tr><td>

```go
type smap struct {
  sync.Mutex // only for unexported types

  data map[string]string
}

func newSMap() *smap {
  return &smap{
    data: make(map[string]string),
  }
}

func (m *smap) Get(k string) string {
  m.Lock()
  defer m.Unlock()

  return m.data[k]
}
```

</td><td>

```go
type SMap struct {
  mu sync.Mutex

  data map[string]string
}

func NewSMap() *SMap {
  return &SMap{
    data: make(map[string]string),
  }
}

func (m *SMap) Get(k string) string {
  m.mu.Lock()
  defer m.mu.Unlock()

  return m.data[k]
}
```

</td></tr>

</tr>
<tr>
<td>インターナルな型やmutexのインタフェースを実装している必要がある場合には埋め込みを使う</td>
<td>公開されている型にはプライベートなフィールドを使う</td>
</tr>

</tbody></table>

## Copy Slices and Maps at Boundaries
スライスやマップは内部でデータへのポインタが含まれています。なのでコピーする際には注意してください。

### Receiving Slices and Maps
引数として受け取ってフィールドに保存したスライスは、他の箇所でデータが書き換わる可能性があることを覚えておいてください。

<table>
<thead><tr><th>Bad</th> <th>Good</th></tr></thead>
<tbody>
<tr>
<td>

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = trips
}

trips := ...
d1.SetTrips(trips)

// ここで値が変わると d1.trips[0] も変わる
trips[0] = ...
```

</td>
<td>

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}

trips := ...
d1.SetTrips(trips)

// d1.trips に変更が及ばない
trips[0] = ...
```

</td>
</tr>

</tbody>
</table>

### Returning Slices and Maps
同じように、公開せずに内部に保持しているスライスやマップが変更されることもあります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Stats struct {
  mu sync.Mutex
  counters map[string]int
}

// Snapshot returns the current stats.
func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  return s.counters
}

// snapshot は mutex で守られない
// レースコンディションが起きる
snapshot := stats.Snapshot()
```

</td><td>

```go
type Stats struct {
  mu sync.Mutex
  counters map[string]int
}

func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  result := make(map[string]int, len(s.counters))
  for k, v := range s.counters {
    result[k] = v
  }
  return result
}

// snapshot はただのコピーなので変更しても影響はない
snapshot := stats.Snapshot()
```

</td></tr>
</tbody></table>

## Defer to Clean Up
ファイルや mutex のロックなどをクリーンアップするために defer を使おう

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
p.Lock()
if p.count < 10 {
  p.Unlock()
  return p.count
}

p.count++
newCount := p.count
p.Unlock()

return newCount

// easy to miss unlocks due to multiple returns
```

</td><td>

```go
p.Lock()
defer p.Unlock()

if p.count < 10 {
  return p.count
}

p.count++
return p.count

// more readable
```

</td></tr>
</tbody></table>

defer のオーバーヘッドは非常に小さいです。
関数の実行時間がナノ秒のオーダーである場合には避ける必要があります。
defer を使ってほんの少しの実行コストを払えば可読性がとてもあがります。
これはシンプルなメモリアクセス以上の計算が必要な大きなメソッドに特に当てはまります。

## Channel Size is One or None
channel のサイズは普段は1もしくはバッファなしのものにするべきです。
デフォルトでは channel はバッファなしでサイズが0になっています。
それより大きいサイズにする場合はよく考える必要があります。
どのようにしてサイズを決定するのか、チャネルがいっぱいになり処理がブロックされたときにどのような挙動をするかよく考える必要があります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// Ought to be enough for anybody!
c := make(chan int, 64)
```

</td><td>

```go
// Size of one
c := make(chan int, 1) // or
// Unbuffered channel, size of zero
c := make(chan int)
```

</td></tr>
</tbody></table>

## Start Enums at One
Go で enum を導入するときの標準的な方法は、型を定義して `const` のグループを作り、初期値を `itoa` にすることです。
変数のデフォルト値はゼロ値です。なので通常はゼロ値ではない値から enum を始めるべきでしょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Operation int

const (
  Add Operation = iota
  Subtract
  Multiply
)

// Add=0, Subtract=1, Multiply=2
```

</td><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)

// Add=1, Subtract=2, Multiply=3
```

</td></tr>
</tbody></table>

ただゼロ値を使うことに意味があるケースもあります。
例えばゼロ値をデフォルトの挙動として扱いたい場合です。

```go
type LogOutput int

const (
  LogToStdout LogOutput = iota
  LogToFile
  LogToRemote
)

// LogToStdout=0, LogToFile=1, LogToRemote=2
```

## Error Types
エラーを定義する方法には様々な種類があります。

* シンプルな文字列と[`errors.New`]( https://golang.org/pkg/errors/#New )で定義する方法
* [`fmt.Errorf`]( https://golang.org/pkg/fmt/#Errorf )を使ってフォーマットされた文字列を使う方法
* Error() メソッドを実装した自前のエラー型を定義する方法
* ["pkg/errors".Wrap]( https://godoc.org/github.com/pkg/errors#Wrap )を使ってラップする方法

エラーを返す場合、以下のことに注意する必要があります。

* このエラーは追加情報が必要ないか？もし無いなら、[`errors.New`]( https://golang.org/pkg/errors/#New )で十分です。
* 利用する側がエラーを検知してハンドリングする必要がありますか？その場合自前で`Error()`メソッドを実装した型を作る必要があります。
* 下流のエラーを更に上流に返していますか？もしそうなら、[section on error wrapping.]( https://github.com/uber-go/guide/blob/master/style.md#error-wrapping )の章を参考にしてください。
* これらに当てはまらないなら、[`fmt.Errorf`]( https://golang.org/pkg/fmt/#Errorf )で問題ないでしょう。

もし利用側がエラーを検出する必要があり、あなたが[`errors.New`]( https://golang.org/pkg/errors/#New )でシンプルなエラーを返している場合、`var`を使って予めパッケージ変数にしておくのがよいでしょう。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open() error {
  return errors.New("could not open")
}

// package bar

func use() {
  if err := foo.Open(); err != nil {
    if err.Error() == "could not open" {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

</td><td>

```go
// package foo

var ErrCouldNotOpen = errors.New("could not open")

func Open() error {
  return ErrCouldNotOpen
}

// package bar

if err := foo.Open(); err != nil {
  if err == foo.ErrCouldNotOpen {
    // handle
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

もし利用側がエラーを検出する必要があり、あなたがエラーに静的な文字列ではなく更に情報を足したい場合、自前の型を定義する必要があります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func open(file string) error {
  return fmt.Errorf("file %q not found", file)
}

func use() {
  if err := open(); err != nil {
    if strings.Contains(err.Error(), "not found") {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

</td><td>

```go
type errNotFound struct {
  file string
}

func (e errNotFound) Error() string {
  return fmt.Sprintf("file %q not found", e.file)
}

func open(file string) error {
  return errNotFound{file: file}
}

func use() {
  if err := open(); err != nil {
    if _, ok := err.(errNotFound); ok {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

</td></tr>
</tbody></table>

自前のエラー型を公開する場合、それも公開APIの一種です。気をつけましょう。
代わりにエラー型と一致しているかを判定する関数を公開するほうがよいでしょう。

```go
// package foo

type errNotFound struct {
  file string
}

func (e errNotFound) Error() string {
  return fmt.Sprintf("file %q not found", e.file)
}

func IsNotFoundError(err error) bool {
  _, ok := err.(errNotFound)
  return ok
}

func Open(file string) error {
  return errNotFound{file: file}
}

// package bar

if err := foo.Open("foo"); err != nil {
  if foo.IsNotFoundError(err) {
    // handle
  } else {
    panic("unknown error")
  }
}
```

## Error Wrapping
エラーを伝搬させるためには以下の3つの方法が主流です。

* 新たに情報を足さない場合、受けたエラーをそのまま返します。
* [`"github.com/pkg/errors".Wrap`]( https://godoc.org/github.com/pkg/errors#Wrap )を使って情報を足します。エラーメッセージはより多くの情報を持てますし、[`"github.com/pkg/errors".Cause`]( https://godoc.org/github.com/pkg/errors#Cause )を使えば元のエラーを取り出すこともできます。
* 呼び出し側がエラーをチェックしたりハンドリングする必要がない場合は[fmt.Errorf]( https://golang.org/pkg/fmt/#Errorf )を使いましょう。

"connection refused"のような曖昧なエラーを返すよりも、"call service foo: connection refused"のようなより役に立つ情報を返しましょう。

返ってきたエラーに情報を足す場合、"failed to"のような表現を省略することでより簡潔になります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "failed to create new store: %s", err)
}
```

</td><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "new store: %s", err)
}
```

<tr><td>

```
failed to x: failed to y: failed to create new store: the error
```

</td><td>

```
x: y: new store: the error
```

</td></tr>
</tbody></table>

ですがエラーメッセージが他のシステムに送られる場合、errタグを付けたりFAILEDプレフィックスをつけたりと、よりエラーメッセージであることを明確にする必要があります。

## Handle Type Assertion Failures
[型アサーション]( https://golang.org/ref/spec#Type_assertions )で1つの戻り値を受け取る場合、その型でなかったらパニックを起こします。
型アサーションではその型に変換できたかを示すbool値も同時に返ってくるので、それで事前にチェックしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
t := i.(string)
```

</td><td>

```go
t, ok := i.(string)
if !ok {
  // 安全にエラーを処理する
}
```

</td></tr>
</tbody></table>

## Don't Panic
プロダクションで動くコードはパニックを避けなければいけません。
パニックは連鎖的障害の主な原因です。
もしエラーが起きた場合、関数はエラーを返して、呼び出し元がどのようにエラーをハンドリングするか決めさせる必要があります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func foo(bar string) {
  if len(bar) == 0 {
    panic("bar must not be empty")
  }
  // ...
}

func main() {
  if len(os.Args) != 2 {
    fmt.Println("USAGE: foo <bar>")
    os.Exit(1)
  }
  foo(os.Args[1])
}
```

</td><td>

```go
func foo(bar string) error {
  if len(bar) == 0 {
    return errors.New("bar must not be empty")
  }
  // ...
  return nil
}

func main() {
  if len(os.Args) != 2 {
    fmt.Println("USAGE: foo <bar>")
    os.Exit(1)
  }
  if err := foo(os.Args[1]); err != nil {
    panic(err)
  }
}
```

</td></tr>
</tbody></table>

`panic`と`recover`はエラーハンドリングではありません。
プログラムはnil参照などの回復不可能な状況が発生したとき以外は出すべきではありません。
ただプログラムの初期化時は例外です。
プログラムが開始するときに異常が起きた場合にはpanicを起こしてもよいでしょう。

```go
var _statusTemplate = template.Must(template.New("name").Parse("_statusHTML"))
```

またテストでは、テストが失敗したことを示すためには`panic`ではなくて `t.Fatal` や `t.FailNow` を使うようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  panic("failed to set up test")
}
```

</td><td>

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  t.Fatal("failed to set up test")
}
```

</td></tr>
</tbody></table>

## Use go.uber.org/atomic 
**TODO: もう少し噛み砕く**

`int32`や`int64`などの変数に対してアトミックな操作をするために[sync/atomic]( https://golang.org/pkg/sync/atomic/ )パッケージが使われます。
ですがこれだとコードの漏れが発生しやすい欠点があります。

[go.uber.org/atomic]( https://godoc.org/go.uber.org/atomic )は実際のデータの型を隠すことにより型安全にこれらの操作を実行することができます。
また、便利な`atomic.Bool`型もあります。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type foo struct {
  running int32  // atomic
}

func (f* foo) start() {
  if atomic.SwapInt32(&f.running, 1) == 1 {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running == 1  // race!
}
```

</td><td>

```go
type foo struct {
  running atomic.Bool
}

func (f *foo) start() {
  if f.running.Swap(true) {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running.Load()
}
```

</td></tr>
</tbody></table>

# Performance
パフォーマンスガイドラインは特によく実行される箇所にのみ適用されます。

## Prefer strconv over fmt
数字と文字列を単に変換する場合、`fmt` パッケージよりも `strconv` パッケージのほうが高速に実行されます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for i := 0; i < b.N; i++ {
  s := fmt.Sprint(rand.Int())
}
```

</td><td>

```go
for i := 0; i < b.N; i++ {
  s := strconv.Itoa(rand.Int())
}
```

</td></tr>
<tr><td>

```
BenchmarkFmtSprint-4    143 ns/op    2 allocs/op
```

</td><td>

```
BenchmarkStrconv-4    64.2 ns/op    1 allocs/op
```

</td></tr>
</tbody></table>

## Avoid string-to-byte conversion
固定の文字列からバイト列を何度も生成するのは避けましょう。
代わりに変数に格納してそれを使うようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for i := 0; i < b.N; i++ {
  w.Write([]byte("Hello world"))
}
```

</td><td>

```go
data := []byte("Hello world")
for i := 0; i < b.N; i++ {
  w.Write(data)
}
```

</tr>
<tr><td>

```
BenchmarkBad-4   50000000   22.2 ns/op
```

</td><td>

```
BenchmarkGood-4  500000000   3.25 ns/op
```

</td></tr>
</tbody></table>

# Style

## Group Similar Declarations
Go は似たような宣言をグループにまとめることができます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import "a"
import "b"
```

</td><td>

```go
import (
  "a"
  "b"
)
```

</td></tr>
</tbody></table>

これはパッケージ定数やパッケージ変数、型定義などにも利用できます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go

const a = 1
const b = 2



var a = 1
var b = 2



type Area float64
type Volume float64
```

</td><td>

```go
const (
  a = 1
  b = 2
)

var (
  a = 1
  b = 2
)

type (
  Area float64
  Volume float64
)
```

</td></tr>
</tbody></table>

関係が近いものだけをグループ化しましょう。
関係ないものまでグループ化するのは避けましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
  ENV_VAR = "MY_ENV"
)
```

</td><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)

const ENV_VAR = "MY_ENV"
```

</td></tr>
</tbody></table>

グループ化を使う場所に制限は無いので、関数内でも使うことができます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func f() string {
  var red = color.New(0xff0000)
  var green = color.New(0x00ff00)
  var blue = color.New(0x0000ff)

  ...
}
```

</td><td>

```go
func f() string {
  var (
    red   = color.New(0xff0000)
    green = color.New(0x00ff00)
    blue  = color.New(0x0000ff)
  )

  ...
}
```

</td></tr>
</tbody></table>

## Import Group Ordering
import のグループは次の2つに分けるべきです。

1. 標準パッケージ
2. それ以外

goimports がデフォルトで適用してくれます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import (
  "fmt"
  "os"
  "go.uber.org/atomic"
  "golang.org/x/sync/errgroup"
)
```

</td><td>

```go
import (
  "fmt"
  "os"

  "go.uber.org/atomic"
  "golang.org/x/sync/errgroup"
)
```

</td></tr>
</tbody></table>

## Package Names
パッケージ名をつける場合、以下のルールに従いましょう。

* 全て小文字で大文字やアンダースコアを使わない
* ほとんどの呼び出し側が名前付きインポートをする必要がないようにする
* 短く簡潔にすること。全ての呼び出し側で識別されることを意識してください
* 複数形にしないこと。`net/urls`ではなく`net/url`です
* "common"、"util"、"shared"、"lib"などを使わないこと。これらはなんの情報もない名前です。

[Package Names]( https://blog.golang.org/package-names )や[Style guideline for Go packages]( https://rakyll.org/style-packages/ )も参考にしてください。

## Function Names
関数名にはGoコミュニティの規則である[MixedCaps]( https://golang.org/doc/effective_go.html#mixed-caps )に従います。
例外はテスト関数です。
`TestMyFunction_WhatIsBeingTested`のようにテストの目的を示すためにアンダースコアを使って分割します。

## Import Aliasing
インポートエイリアスはパッケージ名とパッケージパスの末尾が一致していない場合に利用します。

```go
import (
  "net/http"

  client "example.com/client-go"
  trace "example.com/trace/v2"
)
```

他にもインポートするパッケージの名前がバッティングした場合には使います。
それ以外の場合は使わないようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import (
  "fmt"
  "os"


  nettrace "golang.net/x/trace"
)
```

</td><td>

```go
import (
  "fmt"
  "os"
  "runtime/trace"

  nettrace "golang.net/x/trace"
)
```

</td></tr>
</tbody></table>

## Function Grouping and Ordering

* 関数は呼び出される順番におおまかにソートされるべきです
* 関数はレシーバーごとにまとめられているべきです。

なので、`struct`、`const`、`var`の次にパッケージ外に公開されている関数が来るべきです。

`newXYZ()`や`NewXYZ()`は型が定義されたあと、他のメソッドの前に定義されている必要があります。

関数はレシーバーごとにまとめられているので、ユーティリティな関数はファイルの最後の方に出てくるはずです。


<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func (s *something) Cost() {
  return calcCost(s.weights)
}

type something struct{ ... }

func calcCost(n []int) int {...}

func (s *something) Stop() {...}

func newSomething() *something {
    return &something{}
}
```

</td><td>

```go
type something struct{ ... }

func newSomething() *something {
    return &something{}
}

func (s *something) Cost() {
  return calcCost(s.weights)
}

func (s *something) Stop() {...}

func calcCost(n []int) int {...}
```

</td></tr>
</tbody></table>

## Reduce Nesting
エラーや特殊ケースなどは早めにハンドリングして`return`したりループ内では`continue`や`break`してネストが浅いコードを目指しましょう。
ネストが深いコードを減らしていきましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for _, v := range data {
  if v.F1 == 1 {
    v = process(v)
    if err := v.Call(); err == nil {
      v.Send()
    } else {
      return err
    }
  } else {
    log.Printf("Invalid v: %v", v)
  }
}
```

</td><td>

```go
for _, v := range data {
  if v.F1 != 1 {
    log.Printf("Invalid v: %v", v)
    continue
  }

  v = process(v)
  if err := v.Call(); err != nil {
    return err
  }
  v.Send()
}
```

</td></tr>
</tbody></table>

## Unnecessary Else
if-else のどちらでも変数に代入する場合、条件に一致した場合に上書きするようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var a int
if b {
  a = 100
} else {
  a = 10
}
```

</td><td>

```go
a := 10
if b {
  a = 100
}
```

</td></tr>
</tbody></table>

## Top-level Variable Declarations
パッケージ変数で式と同じ型なら型名を指定しないようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var _s string = F()

func F() string { return "A" }
```

</td><td>

```go
var _s = F()
// Since F already states that it returns a string, we don't need to specify
// the type again.

func F() string { return "A" }
```

</td></tr>
</tbody></table>

式の型と合わない場合は明示するようにしましょう。

```go
type myError struct{}

func (myError) Error() string { return "error" }

func F() myError { return myError{} }

var _e error = F()
// F は myError 型を返すが私達は error 型が欲しい
```

## Prefix Unexported Globals with _
公開されてないtop-levelの`var`や`const`の名前には最初にアンダースコアをつけることでより内部向けてあることが明確になります。
ただ`err`で始まる変数名は例外です。
理由としてtop-levelの変数のスコープはそのパッケージ全体です。一般的な名前を使うと別のファイルで間違った値を使ってしまうことになります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// foo.go

const (
  defaultPort = 8080
  defaultUser = "user"
)

// bar.go

func Bar() {
  defaultPort := 9090
  ...
  fmt.Println("Default port", defaultPort)

  // We will not see a compile error if the first line of
  // Bar() is deleted.
}
```

</td><td>

```go
// foo.go

const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

</td></tr>
</tbody></table>

## Embedding in Structs
mutexなど埋め込まれた型は構造体の定義の最初に置くべきです。
また、通常のフィールドと区別するために1行開ける必要があります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Client struct {
  version int
  http.Client
}
```

</td><td>

```go
type Client struct {
  http.Client

  version int
}
```

</td></tr>
</tbody></table>

## Use Field Names to initialize Structs

構造体を初期化する際にはフィールド名を書くようにしましょう。
[`go vet`]( https://golang.org/cmd/vet/ )でこのルールは指摘されます。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
k := User{"John", "Doe", true}
```

</td><td>

```go
k := User{
    FirstName: "John",
    LastName: "Doe",
    Admin: true,
}
```

</td></tr>
</tbody></table>

例外としてフィールド数が3以下のテストケースなら省略してもよいです。

```go
tests := []struct{
  op Operation
  want string
}{
  {Add, "add"},
  {Subtract, "subtract"},
}
```

## Local Variable Declarations
変数が明示的に設定される場合、`:=` 演算子を利用しましょう。
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var s = "foo"
```

</td><td>

```go
s := "foo"
```

</td></tr>
</tbody></table>

しかし空のスライスを宣言する場合は`var`キーワードを利用したほうがよいでしょう。[参考資料: Declearing Empty Slices]( https://github.com/golang/go/wiki/CodeReviewComments#declaring-empty-slices )

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func f(list []int) {
  filtered := []int{}
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

</td><td>

```go
func f(list []int) {
  var filtered []int
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

</td></tr>
</tbody></table>

## nil is a valid slice
`nil` は長さ0のスライスとして有効です。
つまり以下のものが有効です。

* 長さ0のスライスを返す代わりに`nil`を返す
    <table>
    <thead><tr><th>Bad</th><th>Good</th></tr></thead>
    <tbody>
    <tr><td>

    ```go
    if x == "" {
    return []int{}
    }
    ```

    </td><td>

    ```go
    if x == "" {
    return nil
    }
    ```

    </td></tr>
    </tbody></table>

* スライスが空かチェックするためには`nil`かチェックするのではなく`len(s) == 0`でチェックする 
  <table>
  <thead><tr><th>Bad</th><th>Good</th></tr></thead>
  <tbody>
  <tr><td>

  ```go
  func isEmpty(s []string) bool {
    return s == nil
  }
  ```

  </td><td>

  ```go
  func isEmpty(s []string) bool {
    return len(s) == 0
  }
  ```

  </td></tr>
  </tbody></table>

* varで宣言しただけのゼロ値が有効
  <table>
  <thead><tr><th>Bad</th><th>Good</th></tr></thead>
  <tbody>
  <tr><td>

  ```go
  nums := []int{}
  // or, nums := make([]int)

  if add1 {
    nums = append(nums, 1)
  }

  if add2 {
    nums = append(nums, 2)
  }
  ```

  </td><td>

  ```go
  var nums []int

  if add1 {
    nums = append(nums, 1)
  }

  if add2 {
    nums = append(nums, 2)
  }
  ```

  </td></tr>
  </tbody></table>

## Reduce Scope of Variables
できる限り変数のスコープを減らしましょう。ただネストを浅くすることとバランスを考えてください。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
err := ioutil.WriteFile(name, data, 0644)
if err != nil {
 return err
}
```

</td><td>

```go
if err := ioutil.WriteFile(name, data, 0644); err != nil {
 return err
}
```

</td></tr>
</tbody></table>

もし関数の戻り値をifの外で利用する場合、あまりスコープを縮めようとしなくてもよいでしょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
if data, err := ioutil.ReadFile(name); err == nil {
  err = cfg.Decode(data)
  if err != nil {
    return err
  }

  fmt.Println(cfg)
  return nil
} else {
  return err
}
```

</td><td>

```go
data, err := ioutil.ReadFile(name)
if err != nil {
   return err
}

if err := cfg.Decode(data); err != nil {
  return err
}

fmt.Println(cfg)
return nil
```

</td></tr>
</tbody></table>

## Avoid Naked Parameters
値をそのまま関数の引数に入れることは可読性を損ないます。
もし分かりづらいならC言語スタイルのコメントで読みやすくしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true, true)
```

</td><td>

```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true /* isLocal */, true /* done */)
```

</td></tr>
</tbody></table>

よりよいのはただの`bool`を自作の型で置き換えることです。こうすると型安全ですし可読性も上がります。
更に将来的にtrue/false以外の状態も利用可能に修正することもできます。

```go
type Region int

const (
  UnknownRegion Region = iota
  Local
)

type Status int

const (
  StatusReady = iota + 1
  StatusDone
  // Maybe we will have a StatusInProgress in the future.
)

func printInfo(name string, region Region, status Status)
```

## Use Raw String Literals to Avoid Escaping
Goは複数行や引用符のために[`Raw string literal`]( https://golang.org/ref/spec#raw_string_lit )をサポートしています。
これらをうまく使って手動でエスケープした読みづらい文字列を避けてください。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
wantError := "unknown name:\"test\""
```

</td><td>

```go
wantError := `unknown error:"test"`
```

</td></tr>
</tbody></table>

## Initializing Struct References
構造体の初期化と同じように構造体のポインタを初期化するときは`new(T)`ではなく、`&T{}`を使いましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
sval := T{Name: "foo"}

// inconsistent
sptr := new(T)
sptr.Name = "bar"
```

</td><td>

```go
sval := T{Name: "foo"}

sptr := &T{Name: "bar"}
```

</td></tr>
</tbody></table>

## Format Strings outside Printf
フォーマット用の文字列を`Printf`スタイルの外で定義する場合は`const`を使いましょう。
こうすることで`go vet`などの静的解析ツールでチェックしやすくなります。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
msg := "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

</td><td>

```go
const msg = "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

</td></tr>
</tbody></table>

## Naming Printf-style Functions
`Printf`スタイルの関数を使う場合、`go vet`がフォーマットをチェックできるか確認しましょう。

これは可能であれば`Printf`スタイルの関数名を使う必要があることを示しています。
`go vet`はデフォルトでこれらの関数をチェックします。

事前に定義された関数名を使わない場合、関数名の最後を`f`にしましょう。
例えば`Wrap`ではなく`Wrapf`にします。
`go vet`は特定の`Printf`スタイルのチェックができるようになっていますが、末尾が`f`である必要があります。

```shell
$ go vet -printfuncs=wrapf,statusf
```

[go vet: Printf family check]( https://kuzminva.wordpress.com/2017/11/07/go-vet-printf-family-check/ )を更に参照してください。

# Patterns
## Test Tables
[サブテスト]( https://blog.golang.org/subtests )を利用したテーブルドリブンテストでコアのテストロジックを繰り返すときにコードの重複を避けるようにしましょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func TestSplitHostPort(t *testing.T)

host, port, err := net.SplitHostPort("192.0.2.0:8000")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("192.0.2.0:http")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "http", port)

host, port, err = net.SplitHostPort(":8000")
require.NoError(t, err)
assert.Equal(t, "", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("1:8")
require.NoError(t, err)
assert.Equal(t, "1", host)
assert.Equal(t, "8", port)
```

</td><td>

```go
// func TestSplitHostPort(t *testing.T)

tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  {
    give:     "192.0.2.0:8000",
    wantHost: "192.0.2.0",
    wantPort: "8000",
  },
  {
    give:     "192.0.2.0:http",
    wantHost: "192.0.2.0",
    wantPort: "http",
  },
  {
    give:     ":8000",
    wantHost: "",
    wantPort: "8000",
  },
  {
    give:     "1:8",
    wantHost: "1",
    wantPort: "8",
  },
}

for _, tt := range tests {
  t.Run(tt.give, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.give)
    require.NoError(t, err)
    assert.Equal(t, tt.wantHost, host)
    assert.Equal(t, tt.wantPort, port)
  })
}
```

</td></tr>
</tbody></table>

テストテーブルを使うと、エラーメッセージへの情報の追加やテストケースの追加も簡単ですし、コードも少なくなります。

テストケースのルールはテストケースの構造体のスライス名が `tests` 、ループ内のそれぞれのテストケースの変数名が `tt` とします。
更に入力値と出力値をわかりやすくするために`give`や`want`などのプレフィックスをつけることを推奨しています。

```go
tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  // ...
}

for _, tt := range tests {
  // ...
}
```

## Functional Options
Functional Option パターンは不透明なOption型を使って内部の構造体に情報を渡すパターンです。
可変長引数を受け取り、それらを順に内部のオプションに渡します。

コンストラクタや、公開されたAPIで3つ以上の多くの引数が必要な場合、このパターンを使うと良いでしょう。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// package db

func Connect(
  addr string,
  timeout time.Duration,
  caching bool,
) (*Connection, error) {
  // ...
}

// Timeout and caching must always be provided,
// even if the user wants to use the default.

db.Connect(addr, db.DefaultTimeout, db.DefaultCaching)
db.Connect(addr, newTimeout, db.DefaultCaching)
db.Connect(addr, db.DefaultTimeout, false /* caching */)
db.Connect(addr, newTimeout, false /* caching */)
```

</td><td>

```go
type options struct {
  timeout time.Duration
  caching bool
}

// Option overrides behavior of Connect.
type Option interface {
  apply(*options)
}

type optionFunc func(*options)

func (f optionFunc) apply(o *options) {
  f(o)
}

func WithTimeout(t time.Duration) Option {
  return optionFunc(func(o *options) {
    o.timeout = t
  })
}

func WithCaching(cache bool) Option {
  return optionFunc(func(o *options) {
    o.caching = cache
  })
}

// Connect creates a connection.
func Connect(
  addr string,
  opts ...Option,
) (*Connection, error) {
  options := options{
    timeout: defaultTimeout,
    caching: defaultCaching,
  }

  for _, o := range opts {
    o.apply(&options)
  }

  // ...
}

// Options must be provided only if needed.

db.Connect(addr)
db.Connect(addr, db.WithTimeout(newTimeout))
db.Connect(addr, db.WithCaching(false))
db.Connect(
  addr,
  db.WithCaching(false),
  db.WithTimeout(newTimeout),
)
```

</td></tr>
</tbody></table>

更に以下の資料が参考になります。

* [Self-referential functions and the design of options]( https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html )
* [Functional options for friendly APIs]( https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis )