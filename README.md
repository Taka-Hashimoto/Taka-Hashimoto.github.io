# Hashimoto Lab Website — 開発引き継ぎドキュメント

## プロジェクト概要

橋本研究室（兵庫県立大学 大学院工学研究科 工学専攻 知能情報分野）のWebサイト。
GitHub Pages でホスティングする単一 `index.html` ファイル。
MuJoCo WASM による六足ロボットの3Dシミュレーションデモを埋め込む。

---

## 現在の状態

### 完成済み

- **index.html** — 研究室Webサイト本体（日本語、ダークテーマ、レスポンシブ対応）
  - ナビゲーション（デモ / 紹介 / 研究 / 業績 / メンバー / アクセス）
  - ヒーローセクション（AI × Robotics をメインテーマ）
  - デモセクション — `https://taka-hashimoto.github.io/mujoco_wasm/` をiframe埋め込み
  - メンバー紹介（About）— プロフィール、研究キーワード、各種リンク
  - 研究内容 — ビジョン文のみ（カードは後から追加予定）
  - 研究業績 — タブ切り替え（学術論文8本、国際会議3本、国内発表6件、研究資金2件）
  - メンバー一覧（教員 + 募集中表示）
  - アクセス（Google Maps埋め込み、住所、所属、連絡先、リンク集）
  - フッター

### 未完了（次のステップ）

1. **mujoco_wasm の hexapod 統合**（最優先）
2. **研究内容セクションへのカード追加**（後日）

---

## 次のステップ: mujoco_wasm と hexapod の統合

### 前提

- `Taka-Hashimoto/mujoco_wasm` — zalo/mujoco_wasm の fork（既にGitHub上に存在）
- `Taka-Hashimoto/hexapod-mjx` — PhantomX hexapod の MuJoCo モデルと CPG 実装

### 作業手順

#### 1. mujoco_wasm リポジトリをクローン

```bash
git clone https://github.com/Taka-Hashimoto/mujoco_wasm.git
cd mujoco_wasm
npm install
```

#### 2. hexapod モデルファイルをコピー

hexapod-mjx リポジトリから以下をコピー:

```bash
# hexapod-mjx から MJCF モデルと STL メッシュを取得
git clone https://github.com/Taka-Hashimoto/hexapod-mjx.git /tmp/hexapod-mjx

# モデルファイルをコピー
cp /tmp/hexapod-mjx/hexapod_mjx/phantomx_hexapod.xml mujoco_wasm/assets/scenes/
cp -r /tmp/hexapod-mjx/hexapod_mjx/assets/ mujoco_wasm/assets/scenes/phantomx_assets/
```

※ XML 内のメッシュパスが `assets/xxx.stl` になっている場合、
  コピー先のディレクトリ構造に合わせてパスを調整する。

#### 3. デフォルトモデルを hexapod に変更

`mujoco_wasm/index.html` (または `examples/main.js` 等) で、
デフォルトで読み込むモデルを hexapod に変更:

```javascript
// 変更前（humanoid等）
mujoco.FS.writeFile("/working/model.xml",
  await (await fetch("./assets/scenes/humanoid.xml")).text());

// 変更後
mujoco.FS.writeFile("/working/model.xml",
  await (await fetch("./assets/scenes/phantomx_hexapod.xml")).text());
```

STL メッシュも同様に FS にマウント:

```javascript
// 各 STL ファイルを FS に書き込む
const meshFiles = ['coxa.stl', 'femur.stl', 'tibia.stl', /* ... */];
for (const f of meshFiles) {
  const buf = await (await fetch(`./assets/scenes/phantomx_assets/${f}`)).arrayBuffer();
  mujoco.FS.writeFile(`/working/assets/${f}`, new Uint8Array(buf));
}
```

#### 4. CPG コントローラを JavaScript に移植

hexapod-mjx の `examples/cpg.py` を参考に、JS で CPG を実装:

```javascript
// CPG パラメータ（tripod gait の例）
const CPG = {
  phases: [0, 0.5, 0.5, 0, 0, 0.5],  // L1, R1, L2, R2, L3, R3
  duty: 0.5,
  period: 0.8,   // seconds
  amplitude: {
    coxa: 0.3,   // rad — 前後スイング
    femur: 0.4,  // rad — 持ち上げ
    tibia: 0.3   // rad — 伸ばし
  }
};

function cpgStep(time, ctrl) {
  for (let leg = 0; leg < 6; leg++) {
    const phase = ((time / CPG.period) + CPG.phases[leg]) % 1.0;
    const isSwing = phase >= CPG.duty;

    // 各関節のオフセット計算
    const coxaIdx = leg * 3;
    const femurIdx = leg * 3 + 1;
    const tibiaIdx = leg * 3 + 2;

    if (isSwing) {
      const t = (phase - CPG.duty) / (1 - CPG.duty); // 0→1
      ctrl[coxaIdx]  = CPG.amplitude.coxa * Math.sin(t * Math.PI);
      ctrl[femurIdx] = -CPG.amplitude.femur * Math.sin(t * Math.PI);
      ctrl[tibiaIdx] = CPG.amplitude.tibia * Math.sin(t * Math.PI);
    } else {
      const t = phase / CPG.duty; // 0→1
      ctrl[coxaIdx]  = -CPG.amplitude.coxa * (2 * t - 1);
      ctrl[femurIdx] = 0;
      ctrl[tibiaIdx] = 0;
    }
  }
}

// メインループ内で毎ステップ呼ぶ
function simulationLoop() {
  cpgStep(data.time, data.ctrl);
  mujoco.mj_step(model, data);
  // Three.js で描画更新
}
```

※ 上記は概念的なコード。実際の振幅・位相・関節インデックスは
  `phantomx_hexapod.xml` のアクチュエータ定義と
  `cpg.py` の実装に合わせて調整が必要。

#### 5. GitHub Pages を有効化

```bash
git add -A
git commit -m "Add PhantomX hexapod model with CPG controller"
git push origin main
```

GitHub リポジトリ → Settings → Pages → Deploy from branch: `main` / `/ (root)` → Save

数分後に `https://taka-hashimoto.github.io/mujoco_wasm/` でアクセス可能になる。

#### 6. 研究室 Web サイトでの確認

`index.html` の iframe src が既に `https://taka-hashimoto.github.io/mujoco_wasm/` を
指しているので、上記が完了すれば自動的に hexapod デモが表示される。

---

## 研究室 Web サイトのデプロイ

### 方法A: ユーザーサイト（推奨）

```bash
# リポジトリ名を Taka-Hashimoto.github.io にする
# → https://taka-hashimoto.github.io/ でアクセス可能
git init
git add index.html
git commit -m "Lab website"
git remote add origin git@github.com:Taka-Hashimoto/Taka-Hashimoto.github.io.git
git push -u origin main
```

Settings → Pages → Deploy from branch: `main` / `/ (root)`

### 方法B: プロジェクトサイト

```bash
# リポジトリ名を lab-website 等にする
# → https://taka-hashimoto.github.io/lab-website/ でアクセス可能
git remote add origin git@github.com:Taka-Hashimoto/lab-website.git
```

---

## ファイル構成

```
hashimoto-lab-handoff/
├── README.md          ← このファイル（引き継ぎドキュメント）
└── index.html         ← 研究室 Web サイト本体
```

---

## 技術スタック

- **フロントエンド**: HTML/CSS/JS（フレームワークなし、単一ファイル）
- **フォント**: DM Serif Display, Noto Sans JP, JetBrains Mono（Google Fonts CDN）
- **シミュレーション**: MuJoCo WASM（iframe 埋め込み）
- **ホスティング**: GitHub Pages（静的サイト）

## 関連リポジトリ

| リポジトリ | 用途 |
|---|---|
| `Taka-Hashimoto/Taka-Hashimoto.github.io` | 研究室Webサイト本体（これからデプロイ） |
| `Taka-Hashimoto/mujoco_wasm` | MuJoCo WASM デモ（hexapod統合先） |
| `Taka-Hashimoto/hexapod-mjx` | hexapod MJCFモデル + CPG + RL環境 |

## 関連URL

| URL | 内容 |
|---|---|
| https://github.com/zalo/mujoco_wasm | fork元のmujoco_wasm |
| https://researchmap.jp/Takanori_Hashimoto | researchmap |
| https://www.eng.u-hyogo.ac.jp/ | 兵庫県立大学 工学研究科 |
| https://www.eng.u-hyogo.ac.jp/group/group48/ | 知能数理計算科学研究グループ |

---

## 研究室の概要（Webサイトに掲載済みのビジョン文）

本研究室では、AI とロボットを融合した知能システムの研究に取り組みます。
近年、人工知能の発展によって画像・言語・行動を扱う技術が急速に進展しています。
一方で、実世界で活動するロボットには、周囲の環境を認識し、状況に応じて適切に判断し、
安定して行動することが求められます。

具体的には、ロボットの運動学習・制御、シミュレーションを活用した学習（Sim-to-Real）、
視覚や言語を活用した知能化などを主なテーマとして扱います。
AI 技術を単に使うだけでなく、「なぜその行動ができるのか」を理解できる数理的な視点も重視し、
理論と実装の両面から研究を進めます。
