# cherry-pick 検証：課題・現象・対策

## 課題

ブランチが3つある構成（`main` / `staging` / `branch1`）で、`branch1` の commit n だけを `main` へ cherry-pick して取り込みたい。

- `staging` で変更 a が行われ、commit m としてコミットされている
- `branch1` はその `staging` から切られているため、変更 a を既に含んでいる
- `branch1` では同じ箇所に変更 b が行われ、commit n としてコミットされている

この commit n 単体の cherry-pick が成功するかどうかが課題である。

実例（本リポジトリ、`file.txt` の5行目）：

| ブランチ / コミット | 5行目 |
|---|---|
| `main` base コミット `116fa6f` | `status = a` |
| `staging` commit m `b6af078` | `status = b`（変更 a） |
| `branch1` commit n `ba12c86` | `status = c`（変更 b） |

## 現象

`main` で cherry-pick を実行すると、内容の競合が発生して停止する。コミットは作成されない。

```bash
git checkout main
git cherry-pick branch1
```

実行結果（終了コード 1）：

```
Auto-merging file.txt
CONFLICT (content): Merge conflict in file.txt
error: could not apply ... n: line5 b -> c
```

- `git status` は `UU file.txt`（両者変更による競合）となる
- `file.txt` には競合マーカーが残る

```
<<<<<<< HEAD
status = a
=======
status = c
>>>>>>> n: line5 b -> c
```

- 中間の値 b は、競合のどこにも現れない

## 原因

cherry-pick は、commit n 自身の diff（b → c）を、n の親の状態（b）を基準とする3方向マージで適用する。

ところが `main` の同じ行は a であり、基準の b とも、結果の c とも異なる。基準に対して両側が同じ行を変更しているため、git は自動判定できず、競合として停止する。

つまり、n の diff が前提とする中間状態 b が `main` に存在しないことが、失敗の原因である。

## 対策

1. **先に commit m を cherry-pick し、続けて commit n を cherry-pick する**
   中間状態 b が `main` に揃うため、n の diff（b → c）がそのまま適用でき、競合しない。
2. **あるいは、`branch1` を `main` へ merge する**
   変更 a・b の両方が履歴とともに取り込まれる。
3. **commit n 単体を取り込む場合は、競合を人手で解決する前提とする**
   cherry-pick は自動では完了しないため、運用手順として明示しておく。
