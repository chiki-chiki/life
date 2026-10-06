
### 症状

- ログ上は同じ値に見える
- `if (a !== b)` が想定外にtrueになる

### 原因

- API or tableデータが string で返ってくる
- 内部カウントは number

```
2 !== "2" // true
```

### 確認方法

```
console.log(value, typeof value)
```

### 対策

どちらかに統一する

```
Number(value)String(value)
```

または比較前に正規化

```
const head = Number(table?.[3]?.head ?? 0);
```

---

## 学び（重要）

- 「ログで同じに見える」は信用しない
- 型を見ないとバグは絶対見えない
- `!==` は“値 + 型”比較