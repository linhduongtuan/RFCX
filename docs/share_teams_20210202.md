# 2021/02/02

基本戦略は
1. スペクトログラムのpre-train
2. 明確にラベル付けされていないラベルの勾配を無視
3. 2.のモデルでre-labeling

の３ステージ制で良さそう


## CVが上がるけどLBが落ちる問題

### 今までCVは上がったけどLBは落ちたケース

#### ①negativeのre-labeling
originの3rd stageに閾値0.01: `CV=0.9260 / LB=0.903` → `CV=0.9446 / LB=0.878`
- 本当はpositiveだけどnegativeだと判断されたラベルがある
- ↑をpseudoで見つけられなかった
- すでに付けられたラベルに関してはきちんと学習できていたのでCVはあがった

#### ②positiveラベルの重み付け 

focalのalpha調整: `CV=0.8952 / LB=0.911` → `CV=0.925 / LB=0.869`  
focalのalpha調整+mixup: `CV=0.9346 / LB=0.912` → `CV=0.9412	/ LB=0.868	`
- ラベルづけされているものに過敏に反応するようになってしまう
- ①とどうように本当はpositiveだけどnegativeだと判断されたラベルを認識できなくなる

#### ③mixup lastlayer
最終層でmixupする: `CV=0.9346 / LB=0.912` → `CV=0.9392 / LB=0.907`
- 不明(②と同じと考えてよい？)


### 考察
- negativeラベルに関してre-labelingするのは危険
- やるとしても相当閾値を厳しくするべき(5 foldすべてのモデルの出力が1e-4以下など)
- positiveラベルをブーストするのも良くなさそう


