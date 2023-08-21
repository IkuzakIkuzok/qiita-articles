---
title: 誤差伝播の計算
tags:
  - Python3
private: false
updated_at: '2020-05-21T18:34:55+09:00'
id: 47b7a8a88aba5ec6f1ca
organization_url_name: null
slide: false
---

## はじめに

実験結果の処理などで計算をしていると，誤差を考慮しなければならないことが多々あります。
誤差を含む値を使用して計算した関数値の誤差は
$$e=\sqrt{\sum_i^ne_i^2\left(\frac{\partial f}{\partial x_i}\right)^2}$$
となります(ここでこの式の導出をすることはしません)。

ご覧の通り，これを計算するのは面倒です。
割り算でも偏微分が面倒なのに，指数関数とか対数関数が出てきたらとんでもないことになります。
そこで，「退屈なことをPythonにやらせる」ことにしました。

## 実行環境

- Windows 10
- Python 3.8.3

f-stringを利用しているので3.6以降で動作します。

### ライブラリ

一応numpyを使っていますが，実は必要ないです。
`numpy.sqrt`しか使っていませんが，これは`x**(1/2)`で代替可能です。
なぜわざわざimportしたかと言えば，科学計算でよく出てくる指数関数・対数関数を使えるようにするためです。
copyは標準で入っていると思います。

## ソースコード

```python
import numpy
from copy import copy
from math import inf


def calc_eps():
  a = 4 / 3
  b = a - 1
  c = b + b + b
  return 1 - c

EPS = calc_eps()


def eval_func(func, beta):
  if callable(func):
    return func(beta)

  for i, b in enumerate(beta):
    func = func.replace(f'${{{i+1}}}', str(b))
  return eval(func)


def diff(func, beta, index):
  last_d = 0.0
  d = inf
  err = inf
  last_e = inf
  eps = 1
  step = 1.1  # Step for eps
  v = eval_func(func, beta)

  while abs(last_d - d) > EPS:
    if eps <= EPS:
      break
    beta2 = copy(beta)
    beta2[index] += eps
    tmp = eval_func(func, beta2) - v
    if tmp == 0.0:
      break
    err = abs((last_d - d) / (step - 1))  # Estimate error
    if err > last_e:
      break
    last_e = err
    last_d = d
    d = tmp / eps
    eps /= step
  return last_d


def parse_data(data, err=None):
  beta = [x[0] for x in data] if err is None else data
  err = [x[-1] for x in data] if err is None else err
  return beta, err


def calc_error(func, beta, err):
  E = 0.0
  for i, e in enumerate(err):
    E += (diff(func, beta, i)**2) * (e**2)
  return numpy.sqrt(E)


def calc_with_error(func, data, err=None):
  beta, err = parse_data(data, err)
  return eval_func(func, beta), calc_error(func, beta, err)


def get_digit(value):
  return int(numpy.log10(value) // 1)


def separate_exp(value):
  d = get_digit(value)
  m = value / (10**d)
  return m, d


def roundat(val, digit):
  d = get_digit(val)
  sgn = val / numpy.absolute(val)
  coeff = 10 ** (d - digit)
  val /= coeff
  val = int(val + 0.5 * sgn) * coeff
  return val


def format_value(val, err, show_sign=False):
  if val < 0:
    val = -val
    sgn = '-'
  else:
    sgn = '' if not show_sign else '+'

  s_err, d_err = separate_exp(err)
  s_err = int(round(s_err))
  if s_err >= 10:
    s_err //= 10
    d_err += 1
  d_val = get_digit(val)

  d = d_val - d_err

  s_val = roundat(val, d)
  m = f'{s_val:e}'.split('e')[0] + '0' * d
  width = 1 if d <= 0 else 2 + d
  m = m[:width]

  err = str(s_err)

  if not d_val >= d_err:
    d_val = d_err

  return f'{sgn}{m}({err})E{d_val}'


def main():
  f = '${1} * ${2} / ${3}'
  data = [(25.4, 0.5), (1.56, 0.01), (13.5, 0.2)]

  value, error = calc_with_error(f, data)

  print(f'Value: {value}')
  print(f'Error: {error}')
  print(format_value(value, error))


if __name__ == '__main__':
  main()
```

## 解説

ダラダラと長いので畳んでおきます。

<details><summary>解説</summary><div>

ソースコードを上から順に解説するとわかりにくいので，実際に実行したときの流れに沿って説明します。
ボイラープレート的な部分は省略します。

### 計算機イプシロンの準備

計算機イプシロンの計算方法については[Pythonの計算機イプシロン](https://qiita.com/ikuzak/items/1332625192daab208e22)をご覧ください。

### 関数とデータの準備

関数は

```python
f = '${1} * ${2} / ${3}'
```

のように文字列で指定するか，

```python
f = lambda b: b[0] * b[1] / b[2]
```

のようにPythonの関数としても指定できます。
(もちろん`def`で宣言してもOKです)

使用するデータは

```python
data = [(25.4, 0.5), (1.56, 0.01), (13.5, 0.2)]
```

のように測定値とその誤差をタプルにしたもののリストを使用します。
これは，

```python
val = [25.4, 1.56, 13.5]
err = [0.5, 0.01, 0.2]
```

のように独立させて計算関数に渡すこともできます。

#### 言い訳

もともとは，後から紹介したバラバラの引数の仕様でした。
ところが，実際に使っていると気体定数やアボガドロ定数などの定数を変数に入れたくなりました。
変数として数値をセットで扱うならタプルの方がよかろう，ということでタプルのリストを使用できるようにしました。
互換性の観点から両方の形式を使用できるようになり，結果としてこのような頭の悪いコードになりました。
(独立したリスト方式のメリットも何かあるはずです……)

### `calc_with_error`

関数値とその誤差を返すものです。そのままです。

#### `parse_data`

`err`が`None`であればデータはタプルのリスト，そうでなければ測定値と誤差がそれぞれ`data`と`err`に入っていると解釈していい感じにします。

#### `eval_func`

実際に関数を計算します。
受け取った関数がcallableであれば直接データを渡し，そうでなければ`${i}`の形式で指定された位置に引数を入れて値を求めます。
まじめにトークナイズせずに文字列の置換で処理しているのであまりよろしくはありませんが……

### `calc_error`

誤差を計算する関数です。
i番目の変数で偏微分した値の2乗とi番目の誤差の2乗をかけて順に足しているだけです。

### `diff`

関数`func`を`i`番目の変数で偏微分した値を返します。
偏微分の定義に従って，変数を微小量変化させたときの変化量を調べています。
「微小量」は変数の値に`EPS`(Epsilonのつもり)をかけたものです。

微小変化量を限りなく小さくすると偏微分係数は正確になっていくのですが，有限の桁数しか持たない数値計算ではあるところを境に誤差が大きくなってしまいます。
そこで，推定誤差を計算して，誤差が大きくなってしまった場合には反復を終了しています。

### `format_value`

誤差を含む値を整形する関数です。
例えば，例で示しているコードでは計算値は2.9351111111111114，誤差は0.07471981498926973となりますが，
これを"2.94(7)E0"という文字列に変換します。

まず，負号を取り除きます。
後半で文字列長を使って切り出しをしている部分があるので，先頭に符号があると面倒になります。
ついでに，正の符号を明示するオプション機能もここで実装されています。

```python
m_err, d_err = separate_exp(err)
m_err = int(m_err+0.5)
if m_err >= 10:
    m_err //= 10
    d_err += 1
```

この部分では，誤差を仮数と指数部分に分離し，仮数部分を四捨五入しています。
`separate_exp`は値の仮数と指数をタプルで返す関数です。

`d_val = get_digit(val)`は`val`(計算値)の桁数(指数部分)を取得しています。

#### `get_digit`

引数の指数部分を返す関数です。
基本的には常用対数を整数に丸めればよいのですが，1未満の数では0に向かって丸めるのではなく負の無限大に向かって丸める必要があるので頭が悪そうな見た目になっています。

`d = d_val - d_err`では，計算値のうち何桁分が信頼できるかを計算しています。
その後，計算値を信頼できる桁までで丸め(`s_val = roundat(val, d)`)，指数形式にして仮数の必要な桁だけ切り出しています。
(`m`は"mantissa"(仮数)です)

最後に，符号，仮数，誤差，指数をまとめて文字列として返します。

変数のプレフィックスは，`s`が"significand"(仮数)，`d`が"digits"(桁数)です。

</div></details>

## 実行例

上記のコードで示した25.4(5)×1.56(1)/13.5(2)では

```
Value: 2.9351111111111114
Error: 0.07471981498926973
2.94(7)E0
```

と表示されます。

### 検証

実際に手計算で頑張ると，
$$e=\sqrt{\left(\frac{1.56}{13.5}\right)^2\times0.5^2+\left(\frac{25.4}{13.5}\right)^2\times0.01^2+\left(-\frac{25.4\times1.56}{13.5^2}\right)^2\times0.2^2}=0.0747\cdots$$
となり，Pythonでも正確に計算できていることが確認できました。

## 最後に

ゴリゴリと力業でなんとかしている部分が多くなってしまったので，よりよい書き方等がありましたら教えていただければ幸いです。
