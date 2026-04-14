# DESIGN — devto-articles

設計判断とその理由を記録する。CLAUDE.md には「何をするか」、ここには「なぜそうするか」。

---

## 現在の状態

このファイルは 2026-04-14 に claude-config 規約 (CLAUDE / SESSION / DESIGN の三層) に沿って初期整備された。明示的に文書化された設計判断はまだ蓄積が少ない。

新しい設計判断 (技術選択、構造変更、規約変更等) が発生したら、本ファイルに以下の形式で追記する:

```
## <判断のタイトル>

**判断:** 何を決めたか (1-2 文)。

### 理由
なぜそう決めたか。

### 代替案と棄却理由
| 案 | 棄却理由 |
|---|---|
| ... | ... |

### 関連
関連する CLAUDE.md セクション、commit、過去の事故等への pointer。
```

参考: `~/Claude/claude-config/DESIGN.md`, `~/Claude/twcu-seminar/DESIGN.md`, `~/Claude/lectures/DESIGN.md` 等。
