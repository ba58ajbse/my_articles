# 現在時刻を利用する関数をテストする方法

## はじめに
現在時刻を扱う関数についてGolangでユニットテストを作成する方法を紹介します。
時間は常に変わるものなので、特定の時刻を期待してテストを行うのは難しく感じるかもしれません。
しかし、依存性注入（DI）やグローバル変数を使えば現在時刻を取得する関数のテストがぐっと楽になります。
ほかにも方法があると思いますが参考になれば幸いです。

## テストしたいケース
例えば下記のような感じです。

まず、現在時刻を取得する関数があるとします。
```golang
package mytime

import "time"

func GetCurrentTime() time.Time {
    return time.Now()
}
```
上記を使用する関数があるとします。
```golang
package hoge

func Greet() string {
    now := mytime.GetCurrentTime()
    hour := now.Hour()

    if hour < 12 {
        return "おはようございます！"
    } else if hour < 18 {
        return "こんにちは！"
    } else {
        return "こんばんは！"
    }
}
```
上記のような関数をテストする場合、`GetCurrentTime()`で取得する値を任意の値に設定し、期待した値が返ってくるかをテストしたいと思います。
しかし、`Greet()`内で現在時刻を取得しているためこのままではテスト実行時に任意の値は設定できません。

## 引数に現在時刻を設定する
この方法が最も簡単です。
`Greet()`の引数に現在時刻を渡すようにすればいいです。
```golang
func Greet(now time.Time) string {
    hour := now.Hour()

    if hour < 12 {
        return "おはようございます！"
    } else if hour < 18 {
        return "こんにちは！"
    } else {
        return "こんばんは！"
    }
}
```
こうすれば、下記のようにテストが書けます
```golang
func TestGreet(t *testing.T) {
    cases := map[string]struct {
        now  time.Time
        want string
    }{
        {time.Date(2023, 1, 1, 9, 0, 0, 0, time.UTC), "おはようございます！"},
        {time.Date(2023, 1, 1, 15, 0, 0, 0, time.UTC), "こんにちは！"},
        {time.Date(2023, 1, 1, 19, 0, 0, 0, time.UTC), "こんばんは！"},
    }

    for testName, tt := range cases {
        res := hoge.Greet(tt.now)
        assert.Equal(t, tt.want, res)
    }
}
```
基本的には外から現在時刻を渡すようにしてあげれば簡単に任意の値を設定できます。
この辺りは実践している人も多いかと思いますが、外から渡せない場合などはどうでしょうか？

## グローバル変数の利用
グローバル変数に現在時刻を取得する関数を代入すれば、テストの時に置き換えることができます。
```golang
var greetTime = mytime.GetCurrentTime

func Greet() string {
    now := greetTime()
    hour := now.Hour()

    if hour < 12 {
        return "おはようございます！"
    } else if hour < 18 {
        return "こんにちは！"
    } else {
        return "こんばんは！"
    }
}
```
テストは下記のように書けます。
```golang
func TestGreet(t *testing.T) {
    cases := map[string]struct {
        now  time.Time
        want string
    }{
        {time.Date(2023, 1, 1, 9, 0, 0, 0, time.UTC), "おはようございます！"},
        {time.Date(2023, 1, 1, 15, 0, 0, 0, time.UTC), "こんにちは！"},
        {time.Date(2023, 1, 1, 19, 0, 0, 0, time.UTC), "こんばんは！"},
    }

    for testName, tt := range cases {
        // `greetTime`に任意の時刻を返却する関数を再代入する
        greetTime = func() time.Time {
            return tt.now
        }
        res := hoge.Greet()
        assert.Equal(t, tt.want, res)
    }
}
```
こうすることで、`Greet()`内で実行される`greetTime()`の返り値は`tt.now`の値が返却されるようになるので任意の値でテストが実行されます。
比較的簡単に実装できますが、グローバル変数を使用しているのでその扱いには注意してください！

## モックを使用する
レイヤードアーキテクチャを使用している場合などにこの方法が使用できます。  
例えばユースケース層に`Greet()`があるとします。
```golang
package usecase

type GreetUsecase struct {
    repo repository.GreetRepository
}

func NewGreetUsecase(repo repository.GreetRepository) *GreetUsecase {
    return &GreetUsecase{
        repo: repo
    }
}

func (u *GreetUsecase) Greet() string {
    now := mytime.GetCurrentTime()
    hour := now.Hour()

    if hour < 12 {
        return "おはようございます！"
    } else if hour < 18 {
        return "こんにちは！"
    } else {
        return "こんばんは！"
    }
}
```
このような場合に、テスト時に`myytime.GetCurrentTime()`をモックして任意の値を返すようにしてみます。  
`GetNow`メソッドを持つインターフェースを作成し、これを用いて時刻を取得するようにします。
```golang
package mytime

import "time"

type MyTime struct {}

type TimeIF interface {
	GetNow() time.Time
}

func NewTime() *Time {
	return &MyTime{}
}

func (t *MyTime)GetNow() time.Time {
	return GetCurrentTime()
}

func GetCurrentTime() time.Time {
    return time.Now()
}
```

先ほどの`Greet()`では、`GreetUsecase`の構造体に定義した`time`経由で現在時刻を取得するようにします。
```go
package usecase

import "mytime"

type GreetUsecase struct {
    repo repository.GreetRepository,
    time mytime.TimeIF,
}

func NewGreetUsecase(repo repository.GreetRepository, time mytime.TimeIF) *GreetUsecase {
    return &GreetUsecase{
        repo: repo,
        time: time,
    }
}

func (u *GreetUsecase) Greet() string {
    now := u.time.GetNow()
    hour := now.Hour()

    if hour < 12 {
        return "おはようございます！"
    } else if hour < 18 {
        return "こんにちは！"
    } else {
        return "こんばんは！"
    }
}
```

テストでは下記のようにモックを作成し、`NewGreetUsecase()`の時にモック関数を定義するようにします。  
`timeMock.On("GetNow").Return(tt.now)`によって、`GetNow`が呼ばれたら、事前に定義した`tt.now`の値を返却するという振る舞いをモックします。
```golang
type timeMock struct {
    mock.Mock
}

func (_m *timeMock) GetNow() {
    res := m.Called()
	return res.Get(0).(time.Time)
}
func TestGreet(t *testing.T) {
    cases := map[string]struct {
        now  time.Time
        want string
    }{
        {time.Date(2023, 1, 1, 9, 0, 0, 0, time.UTC), "おはようございます！"},
        {time.Date(2023, 1, 1, 15, 0, 0, 0, time.UTC), "こんにちは！"},
        {time.Date(2023, 1, 1, 19, 0, 0, 0, time.UTC), "こんばんは！"},
    }

    for testName, tt := range cases {
        timeMock := new(TimeMock)
        timeMock.On("GetNow").Return(tt.now)

        greetUsecase := usecase.NewGreetUsecase(nil, timeMock)

        res := greetUsecase.Greet()
        assert.Equal(t, tt.want, res)
    }
}
```
