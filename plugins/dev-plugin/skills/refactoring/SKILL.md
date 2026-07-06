---
name: refactoring
description: |
  モジュール分割・ファイル分割を伴うリファクタリングを行う際の原則と手順。
  実際の失敗（pb_utils.py 分割、2026-06）から抽出した、繰り返してはいけない
  過ちと守るべき判断基準をまとめている。Python パッケージ分割に特有の
  技術的落とし穴（相対インポート・__all__・循環インポート）も含む。
---

# モジュール分割・リファクタリング ガイド

## このスキルを読む前に

このスキルは、`pb_utils.py`（4200行超）の分割作業（2026年6月）で発生した問題を
出発点にしている。作業自体は「インポートが通る」状態まで到達したが、ユーザーは
**意図しないコード修正が複数発生した**こと、および**グルーピングが役割ベースでなく
処理の技術的類似性に基づいていた**ことを理由にリバートした。

このスキルを読めば、次回の作業で同じ失敗を繰り返さずに済む。

---

## 鉄則 1：リファクタリング中はバグを直さない

リファクタリングのスコープは「コードを動かしたまま構造を変える」こと。

動作中に気づいたバグ（例：`CollisionPair` の tuple unpacking が潜在的に壊れているなど）
は**その場で直してはいけない**。理由は2つある：

1. 修正がリバートされたとき、バグ修正も一緒に失われる
2. 「どこが変わったのか」の追跡が困難になり、デバッグ効率が下がる

**やること**：気づいたバグは別 issue として記録する。リファクタリング PR とは分離する。

---

## 鉄則 2：設計への合意を得てから実装する

作業の順番は必ず次の順番を守る：

```
1. 設計案を提示する（どのファイルに何を置くか、依存関係はどうなるか）
2. ユーザーから合意を得る
3. 実装する
4. 動作確認する
5. ユーザーに操作を返す（次ファイルへ進む前）
```

「まとめてやっていい」とユーザーが言っても、各ファイルの完成後に必ず
動作確認を挟む。一括実装は「どのファイルで何が壊れたか」の切り分けを
不可能にする。

---

## 鉄則 3：グルーピングは「役割」で判断する

判断基準そのものは言語やプロジェクトを問わず適用できる。以下の例は
実際に起きた失敗（Python/PyBulletプロジェクト）のものだが、対象プロジェクトの
言語・ドメインに読み替えて使う。

### 何が「処理類似性」で、何が「役割」か

今回の失敗は、関数を「何をやっているか（アルゴリズム）」で分類したことにある。

| 処理類似性でグルーピング（悪い例） | 役割でグルーピング（良い例） |
|---|---|
| 「AABB 計算はすべて geometry.py へ」 | 「衝突検出に使う AABB 取得は collision.py と同じ文脈にある」 |
| 「カメラ関連はすべて camera.py へ」 | 「カメラ行列の純粋計算（依存なし）は pb_math.py でよい」 |
| 「ジョイント操作は joints.py へ」 | 「`get_difference_fn` は collision.py が使うので、その手前に置く」 |

### 役割で判断するための問い

関数をどのファイルに置くか迷ったとき、次を自問する：

1. **誰が使うか** — 呼び出し側のファイルはどこか。呼び出し元と同じ層か1層上に置く
2. **何に依存するか** — PyBullet を呼ぶか、numpy だけか。依存の性質がファイル帰属を決める
3. **いつ使われるか** — 初期化時か、最適化ループ中か、デバッグ時か

```
例：get_buffered_aabb
  依存：PyBullet あり（get_aabb を呼ぶ）
  使用者：collision.py（衝突判定のためのバッファ取得）
  → geometry.py に置くのは「形状計算」としては正しいが、
    collision.py と同じファイルか、その直接の依存先に置く方が自然
```

### 対象プロジェクトの設計文書を先に読む

対象プロジェクトに、今回の分割方針を記録した設計文書（`docs/design/` 配下など）が
存在する場合は、グルーピング判断を始める前に必ず読む。存在しない場合は、
作業前にユーザーへ設計方針を確認する（鉄則2）。

---

以下の技術的注意点1〜4は Python のパッケージ分割に特有の落とし穴であり、
対象プロジェクトが Python でない場合は適用対象外（読み飛ばしてよい）。
鉄則1〜3は言語を問わず適用する。

## 技術的注意点 1：相対インポートの深さ

Python の相対インポートの `.` の数はパッケージ階層から**計算**する。勘では決めない。

```
tampura_environments/           # パッケージルート
  panda_utils/                  # depth 1
    pb_math.py
    pybullet_mod/               # depth 2
      body/                     # depth 3
        body.py     → ...pb_math = panda_utils.pb_math  ✓ (3ドット)
        joints.py   → ...pb_math = panda_utils.pb_math  ✓ (3ドット)
      planning/                 # depth 3（body/ と同じ深さ）
        collision.py → ...pb_math = panda_utils.pb_math ✓ (3ドット)
```

**実際に発生したミス**：`planning/` と `perception/` のファイルで `....pb_math`（4ドット）を
使っていた。4ドットは `tampura_environments` を指すため `tampura_environments.pb_math` を
探してしまい `ModuleNotFoundError` になった。

**計算方法**：ファイルのパッケージパスを数える。
`panda_utils.pybullet_mod.planning.collision` → ドット4つ分のパッケージがある →
`panda_utils` に戻るには 3 つ遡る → `...pb_math`。

---

## 技術的注意点 2：`__all__` は re-export ハブを壊す

モジュールに `__all__` を定義すると、`from module import *` で公開されるシンボルが
そのリストだけに制限される。これは re-export ハブ（`pb_utils.py` の `from .joints import *` など）
に影響する。

**実際に発生したミス**：`joints.py` に循環インポート対策として以下を追加した：

```python
# joints.py の意図（循環防止）
__all__ = ["get_joints", "get_links", "get_num_joints", "get_all_links"]
```

この結果、`pb_utils.py` の `from .pybullet_mod.body.joints import *` が
4 関数しかエクスポートしなくなり、`control_joints`・`link_from_name` など
13 関数が `pbu.xxx` でアクセスできなくなった。

**正しい対処**：循環インポートは `__all__` で防がない。関数を **明示的に** インポートする
（`from .body import get_joints, get_links` のように）か、関数の配置場所を変える。

---

## 技術的注意点 3：循環インポートの解決は文書化する

循環インポートを避けるために「本来の置き場所でない場所」に関数を置いた場合、
**必ずコメントでその理由を書く**。

```python
# joints.py 冒頭

# get_num_joints・get_joints・get_links・get_all_links は本来 joints.py の管轄だが、
# body.py が joints.py に依存するため、body.py に置いて joints.py が body.py から import する
# 一方向依存（body → types のみ）を維持するための配置
from .body import get_num_joints, get_joints, get_links, get_all_links
```

コメントがないと、次のセッションが「joints.py に置くべきでは？」と判断して移動し、
循環インポートを再発させる。

---

## 技術的注意点 4：新しいファイルを作ったら元のファイルで使われていた全シンボルを確認する

ファイルを分割した後、元のファイルが呼び出されていた全シンボルが
新しい構造から到達可能かを確認する。

**検証の流れ**（この順で行う）：

以下は tampura_environments での具体例。モジュール名・パス（`tampura_environments`・
`panda_utils`・`pb_utils`・`pbu` など）は対象プロジェクトのものに置き換えて実行する。

```bash
# Step 1: import が通るか
python3 -c "from tampura_environments.panda_utils import pb_utils; print('OK')"

# Step 2: 呼び出し元が使っている全シンボルが存在するか
grep -rh "pbu\." tampura_environments/ --include="*.py" | grep -oP "pbu\.\w+" | sort -u \
  > /tmp/symbols.txt
python3 -c "
import tampura_environments.panda_utils.pb_utils as pbu
missing = [s.split('.')[1] for s in open('/tmp/symbols.txt').read().split('\n')
           if s and not hasattr(pbu, s.split('.')[1])]
print('MISSING:', missing)
"

# Step 3: 全環境モジュールが import できるか
python3 -c "
for mod in ['tampura_environments.panda_utils.robot',
            'tampura_environments.panda_utils.primitives',
            'tampura_environments.panda_utils.grasping',
            'tampura_environments.panda_utils.panda_env_utils',
            'tampura_environments.class_uncertain.env',
            'tampura_environments.find_dice.env',
            'tampura_environments.find_block_stack_dice.env',
            'tampura_environments.puck_slide.env',
            'tampura_environments.tool_use.env']:
    try:
        __import__(mod); print('OK:', mod)
    except Exception as e:
        print('FAIL:', mod, e)
"
```

**import が通っても安心しない**。Step 2 を必ず実行する。
`import` 成功 ≠ `pbu.link_from_name` などのシンボルアクセス成功。

---

## よくある見落とし

ファイル分割時に元のモノリスにあった関数が新しいどのファイルにも移されていない
ことがある。特に以下の種類の関数が抜けやすい：

| パターン | 例 |
|---|---|
| 薄いラッパー（1〜3行） | `get_model_info`（`INFO_FROM_BODY` の lookup だけ） |
| コントローラ生成器 | `joint_controller`（`waypoint_joint_controller` と似ているが別物） |
| 純粋計算だが小さい | `get_field_of_view`（camera_matrix から FoV を計算） |
| 1箇所しか使われないユーティリティ | `empty_sequence`、`cached_fn` など |

**対策**：分割完了後に Step 2 の MISSING チェックを実行し、0件になることを確認する。

---

## まとめ：作業チェックリスト

リファクタリング作業を開始する前に確認する：

- [ ] 対象プロジェクトに設計文書（`docs/design/` 配下など）があれば読んだ
- [ ] ユーザーと分割方針の合意を得た
- [ ] 1ファイルごとに実装・確認・ユーザーへの報告を行う計画にした

各ファイル実装後に確認する：

- [ ] import が通る
- [ ] 追加・変更したファイルに**リファクタリング以外の修正**が含まれていない
- [ ] 循環インポートを避けるために関数を移動した場合、コメントを書いた
- [ ] 相対インポートのドット数をパッケージ階層から計算した

全ファイル完成後に確認する：

- [ ] Step 2（MISSING チェック）が 0件
- [ ] 全環境モジュールの import が成功
- [ ] ユーザーに操作を返し、実際の環境で動作確認してもらった
