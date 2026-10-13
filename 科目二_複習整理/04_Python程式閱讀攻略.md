# 科目二 Python 程式閱讀攻略

目標是看懂短程式的**資料範圍、輸出形狀與計算分母**。先讀 [核心重點總整理](./01_重點總整理.md) 的 Python 與資料洩漏段落，再用本頁練習；工具清單與優先級依 [真題校準](./00_真題校準與命題優先級.md)，範例取材自第四至六回模擬題的考點，並非官方原題。

## 一、讀題固定走六步

| 順序 | 先確認什麼 | 常見誤判 |
|---|---|---|
| 1. 輸入 | 每列代表交易、會員還是一天？欄位型別、有無缺失、是否已排序？ | 把單筆交易當成會員總額 |
| 2. 輸出 | 要數值、Series、DataFrame、標籤，還是圖？保留幾列？ | `transform` 與聚合結果形狀混淆 |
| 3. 篩選 | 哪個條件留下哪些列？條件對欄位還是索引？ | `'plan' == 'Pro'` 只比較兩個字串 |
| 4. 分組與分母 | 按誰分組？問總額、平均、有效筆數還是全部列數？ | `count()` 忽略缺失，`size()` 包含缺失 |
| 5. 索引與連接 | `loc` 看標籤、`iloc` 看位置；merge 的鍵有沒有重複？ | left join 不保證維持左表列數 |
| 6. 學參數與時間 | 每個 `fit` 用了哪些資料？特徵在預測當下可得嗎？ | 測試集重新 fit、rolling 包含當日答案 |

可把鏈式呼叫拆成「先做什麼 → 剩下什麼 → 再算什麼」。例如 `groupby('id')['sales'].sum().nlargest(3)` 是先按 id 算總額，再從**各組總額**取前三名。

## 二、Pandas：先看列數，再看值

### 篩選、索引、分組

| 語法 | 讀法 |
|---|---|
| `df['x']`／`df[['x']]` | 前者通常是 Series；後者是單欄 DataFrame |
| `df.loc[mask, 'x']` | 按布林條件篩列，再選 x 欄 |
| `df.loc[label, 'x']` | 按列索引標籤取值；不是按某欄內容篩選 |
| `df.iloc[0, 1]` | 第 1 列、第 2 欄；從 0 起算 |
| `(df['x'] > 0) & (df['y'] == 'A')` | 逐列且條件；每個比較式加括號，或條件用 `\|` |
| `groupby('g')['x'].sum()` | 每組一個結果；通常以 g 為索引 |
| `groupby('g')['x'].transform('sum')` | 把每組總額回填到原有列；列數與順序維持 |
| `sort_values(...).drop_duplicates('id', keep='last')` | 保留排序後每個 id 的最後一列；「最新」須先按版本或時間排序 |

```python
import pandas as pd
df = pd.DataFrame({'id': ['A', 'B', 'A'], 'sales': [10, 40, 20]},
                  index=[100, 200, 300])
print(df.loc[df['id'] == 'A', 'sales'].mean())
print(df.loc[200, 'sales'], df.iloc[0, 1])
print(df.groupby('id')['sales'].sum().nlargest(1).to_dict())
print(df.groupby('id')['sales'].transform('sum').tolist())
```

輸出：

```text
15.0
40 10
{'B': 40}
[30, 40, 30]
```

`loc` 的標籤切片通常包含終點；`iloc` 的位置切片不包含終點。不要把索引的 200 解讀成「第 200 列」。

### NaN／缺失值、型別與分母

| 語法 | 必會判斷 |
|---|---|
| 數值 `s.describe()` | count、mean、std、min、25%、50%、75%、max；50% 是中位數，count 不含缺失 |
| `isna().sum()` | True 當 1 加總，得到缺失數；DataFrame 預設逐欄計算 |
| `count()`／`size()`／`len()` | count 算非缺失值；Series.size 與 len 算全部元素；groupby.size 算每組列數 |
| `mean()`／`sum()` | Pandas 預設略過缺失；平均的分母是有效值數，不能把缺失當 0 |
| `pd.to_numeric(s, errors='coerce')` | 無法解析的文字變成缺失 |
| `s.where(條件)` | 原有位置保留；不符合條件的值預設變成缺失 |
| `astype('Int64')` | 可缺失整數，大小寫有意義；不是一般 `int64` |

```python
import pandas as pd
s = pd.Series(['120', 'NA', '-5', '80'])
v = pd.to_numeric(s, errors='coerce').where(lambda x: x > 0)
print(int(v.isna().sum()), int(v.count()), v.size)
print(float(v.mean()))
year = pd.Series([2024.0, None]).astype('Int64')
print(str(year.dtype), year.isna().tolist())
```

輸出：

```text
2 2 4
100.0
Int64 [False, True]
```

補充兩個邊界：`groupby` 預設不把缺失的**分組鍵**列成群組，需要時用 `dropna=False`；`sum()` 預設對全缺失 Series 得到 0，若要求「沒有有效值就保留缺失」可用 `sum(min_count=1)`。

### merge：共同的鍵不等於一對一

inner 保留兩表都有的鍵；left 保留左表的鍵，右表未匹配欄位填缺失；outer 保留任一表的鍵。**每個鍵若在左表有 m 列、右表有 n 列，匹配會產生 m×n 列。**

```python
import pandas as pd
orders = pd.DataFrame({'id': [1, 2], 'amount': [100, 200]})
contacts = pd.DataFrame({'id': [1, 2, 2], 'channel': ['sms', 'sms', 'email']})
joined = orders.merge(contacts, on='id', how='left')
print(joined['amount'].tolist())
print(len(joined), int(joined['amount'].sum()))
```

輸出：

```text
[100, 200, 200]
3 500
```

若右表理應每個 id 只有一列，`validate='many_to_one'` 可檢查基數；上例會因右表重複鍵而報錯。去重前要按業務規則選主要聯絡資料，不能為了湊列數任意刪除。`concat` 是串接，沒有自動依 id 關聯的語意。Pandas merge 兩側的缺失鍵也會匹配，與常見 SQL NULL 連接行為不同。

### 分批讀取：合併總額與有效筆數

`read_csv(..., chunksize=k)` 每次提供最多 k 列的一批 DataFrame；最後一批可能不足 k 列。`nrows=k` 只讀部分列。全檔平均應是 `各批總額的和 / 各批有效筆數的和`，直接平均每批平均數會忽略批次大小與缺失比例。見自測第 7 題。

## 三、Scikit-learn：標出每個 fit 的資料範圍

`fit` 學參數；`transform` 用已學參數轉換；`fit_transform` 兩者都做；分類模型 `predict(X_test)` 產生預測標籤，再與 `y_test` 比較。**先切分、只在訓練資料 fit**，測試資料沿用參數。

```python
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline
from sklearn.neighbors import KNeighborsClassifier
scaler = StandardScaler().fit([[0.0], [2.0]])
print(scaler.mean_.tolist(), scaler.scale_.tolist())
print(scaler.transform([[4.0]]).tolist())
pipe = make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=1))
pipe.fit([[0.0], [2.0]], [0, 1])
print(pipe.predict([[4.0]]).tolist())
```

輸出：

```text
[1.0] [1.0]
[[3.0]]
[1]
```

StandardScaler 預設以訓練資料 `ddof=0` 的標準差縮放；本例平均 1、標準差 1，所以新值 4 變成 3。結果不限制在 0～1；Pandas `Series.std()` 預設 `ddof=1`，手算時要看清楚。

Pipeline 把前處理與模型串在同一流程。交叉驗證時應把**尚未 fit 的 Pipeline 與原始 X、y**交給驗證工具，每折只用該折訓練部分學縮放、填補、特徵選擇等參數。先對全部 X、y 執行 `SelectKBest.fit_transform` 再交叉驗證，驗證折標籤已影響特徵選擇；事後包 Pipeline 不能補救先前洩漏。

### 混淆矩陣與 F1

`confusion_matrix(y_true, y_pred, labels=[0, 1])` 的**列是真實、欄是預測**：`[[TN, FP], [FN, TP]]`；只有在這個二元順序下，`.ravel()` 才依序得到 `tn, fp, fn, tp`。

```python
from sklearn.metrics import confusion_matrix, f1_score
truth = [1, 1, 1, 0, 0, 0, 0, 0]
pred = [1, 0, 0, 1, 0, 0, 0, 0]
print(confusion_matrix(truth, pred, labels=[0, 1]).tolist())
print(round(f1_score(truth, pred, pos_label=1), 3))
print(round(f1_score(truth, pred, average='weighted'), 3))
```

輸出：

```text
[[4, 1], [2, 1]]
0.4
0.605
```

正類 F1=`2TP/(2TP+FP+FN)`，沒有 TN。weighted F1 是**先各自算各類 F1，再以真實標籤的 support 加權**；上例類別 0 有 5 筆、類別 1 有 3 筆，所以 `(5×8/11+3×0.4)/8≈0.605`。macro 是各類等權；weighted 分數較高仍可能掩蓋少數類問題。

## 四、時間序列：今天早上不能看到今天結束的銷售

建立「前 3 天平均」時，資料須先按門市與日期排序，且每日一列；每個門市內先 `shift(1)` 排除當日，再 `rolling(3).mean()`。若日期缺漏，3 列未必等於 3 天。

```python
import pandas as pd
df = pd.DataFrame({'store': ['A'] * 4 + ['B'] * 4,
                   'sales': [10, 20, 30, 40, 100, 200, 300, 400]})
history = df.groupby('store')['sales'].transform(lambda s: s.shift(1).rolling(3).mean())
print(history.fillna(-1).tolist())  # 僅為列印，用 -1 代表缺少完整歷史
```

輸出：

```text
[-1.0, -1.0, -1.0, 20.0, -1.0, -1.0, -1.0, 200.0]
```

walk-forward 是用過去訓練、較晚資料驗證，再把切分點向前移動；每輪前處理也只 fit 當輪訓練資料。`TimeSeriesSplit` 是常見的擴張訓練視窗方法；有標籤延遲或跨期觀測時，還要按情境設計 gap 與資料可得時間。

## 五、SciPy／Seaborn：辨認函式與參數的意義

| 呼叫 | 考場讀法 |
|---|---|
| `stats.poisson.pmf(k, mu)` | Poisson 平均次數 mu 下，恰好 k 次的機率 |
| `stats.binom.cdf(k, n, p)` | 固定 n 次、成功機率 p 時，至多 k 次的機率；包含 k |
| `1 - dist.cdf(k-1)` | 非負整數計數分布至少 k 次；`1-cdf(k)` 是超過 k 次 |
| `stats.ttest_ind(a, b, equal_var=False)` | Welch 獨立樣本平均數檢定；不是檢定變異數相同 |
| `stats.ttest_rel(before, after)` | 同一批對象的成對平均差檢定 |
| `sns.histplot(data=df, x='x')` | 看 x 的分布；不是畫 x 隨時間變化 |
| `sns.scatterplot(data=df, x='x', y='y', hue='g')` | x、y 關係，g 用顏色區分群組 |
| `sns.heatmap(df.corr(numeric_only=True))` | 視覺化數值欄相關矩陣；相關不代表因果 |
| `sns.barplot(..., estimator='mean')`／`sns.countplot(...)` | 前者依類別估計數值平均；後者計算類別筆數 |

Seaborn 的長條高度要看 `estimator`；不能見到 barplot 就認定是總額。下例明確關閉誤差線、指定類別順序，只核對長條代表的數值：

```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
df = pd.DataFrame({'group': ['A', 'A', 'B'], 'value': [10, 30, 100]})
ax = sns.barplot(data=df, x='group', y='value', order=['A', 'B'],
                 estimator='mean', errorbar=None)
print([float(bar.get_height()) for bar in ax.patches])
plt.close()
```

輸出：

```text
[20.0, 100.0]
```

## 六、12 道短程式自測

每題先在 30～60 秒內寫輸出及一句理由，再展開答案。各段可獨立執行；輸出型別以程式中的 `int`、`float`、`tolist`、`round` 明確化，不考排版或顯示精度。

### 1. 索引標籤還是位置？

```python
import pandas as pd
s = pd.Series([10, 20, 30], index=[100, 200, 300])
print(s.loc[200], s.iloc[0])
print(s.loc[100:200].tolist(), s.iloc[0:1].tolist())
```

<details><summary>答案與理由</summary>

```text
20 10
[10, 20] [10]
```

loc 按標籤且切片包含終點；iloc 按位置且切片不含終點。

</details>

### 2. 先篩選，再算平均

```python
import pandas as pd
df = pd.DataFrame({'plan': ['Pro', 'Free', 'Pro'], 'fee': [10, 99, 30]})
mask = (df['plan'] == 'Pro') & (df['fee'] > 10)
print(df.loc[mask, 'fee'].tolist())
print(float(df.loc[df['plan'] == 'Pro', 'fee'].mean()))
```

<details><summary>答案與理由</summary>

```text
[30]
20.0
```

第一行同時滿足兩條件，只留下 30；第二行只篩 Pro，平均為 (10+30)/2。

</details>

### 3. 聚合與回填的列數

```python
import pandas as pd
df = pd.DataFrame({'g': ['A', 'B', 'A'], 'v': [2, 5, 3]})
print(df.groupby('g', sort=True)['v'].sum().to_dict())
print(df.groupby('g')['v'].transform('sum').tolist())
```

<details><summary>答案與理由</summary>

```text
{'A': 5, 'B': 5}
[5, 5, 5]
```

sum 聚合有兩組；transform 對三筆原始資料回填各組總額。

</details>

### 4. count 與 size 的分母

```python
import pandas as pd
df = pd.DataFrame({'g': ['A', 'A', 'B'], 'v': [10, None, 30]})
print(df.groupby('g', sort=True)['v'].count().tolist())
print(df.groupby('g', sort=True).size().tolist())
```

<details><summary>答案與理由</summary>

```text
[1, 1]
[2, 1]
```

A 組兩列，只有一個有效 v；count 算非缺失，size 算全部列。

</details>

### 5. 兩邊都重複的 merge

```python
import pandas as pd
left = pd.DataFrame({'id': [1, 1], 'amount': [10, 20]})
right = pd.DataFrame({'id': [1, 1], 'tag': ['x', 'y']})
out = left.merge(right, on='id', how='inner')
print(len(out), int(out['amount'].sum()))
```

<details><summary>答案與理由</summary>

```text
4 60
```

同一鍵兩邊各 2 列，產生 2×2 列；10 與 20 各出現兩次。

</details>

### 6. 不合法金額與平均

```python
import pandas as pd
s = pd.to_numeric(pd.Series(['10', '?', '0', '30']), errors='coerce')
valid = s.where(s > 0)
print(int(valid.count()), float(valid.mean()))
```

<details><summary>答案與理由</summary>

```text
2 20.0
```

問號轉成缺失，0 不滿足條件也變缺失；有效值只有 10、30。

</details>

### 7. 分批平均不是平均各批平均

```python
from io import StringIO
import pandas as pd
csv = StringIO('amount,tag\n10,a\n,b\n30,c\n50,d\n')
total, n, sizes = 0.0, 0, []
for part in pd.read_csv(csv, chunksize=2):
    sizes.append(len(part))
    total += part['amount'].sum()
    n += part['amount'].count()
print(sizes, int(n), float(total / n))
```

<details><summary>答案與理由</summary>

```text
[2, 2] 3 30.0
```

四列分兩批，有效金額三筆；(10+30+50)/3=30。各批平均為 10、40，直接平均得到 25 是錯的。

</details>

### 8. 新資料沿用訓練縮放參數

```python
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler().fit([[2.0], [6.0]])
print(float(scaler.transform([[10.0]])[0, 0]))
print(scaler.mean_.tolist())
```

<details><summary>答案與理由</summary>

```text
3.0
[4.0]
```

訓練平均 4、母體型標準差 2，(10−4)/2=3；transform 不重學新資料的平均。

</details>

### 9. 混淆矩陣與加權方向

```python
from sklearn.metrics import confusion_matrix, f1_score
truth = [0, 0, 0, 0, 1, 1]
pred = [0, 0, 0, 1, 0, 1]
print(confusion_matrix(truth, pred, labels=[0, 1]).ravel().tolist())
print(round(f1_score(truth, pred, average='weighted'), 3))
print(round(f1_score(truth, pred, average='macro'), 3))
```

<details><summary>答案與理由</summary>

```text
[3, 1, 1, 1]
0.667
0.625
```

依序 TN、FP、FN、TP。類別 0 的 F1 為 0.75、類別 1 為 0.5；weighted=(4×0.75+2×0.5)/6，macro=(0.75+0.5)/2。

</details>

### 10. walk-forward 的訓練與驗證位置

```python
from sklearn.model_selection import TimeSeriesSplit
X = [[0], [1], [2], [3], [4], [5]]  # 已按時間由早到晚排序
for train, valid in TimeSeriesSplit(n_splits=2, test_size=2).split(X):
    print(train.tolist(), valid.tolist())
```

<details><summary>答案與理由</summary>

```text
[0, 1] [2, 3]
[0, 1, 2, 3] [4, 5]
```

訓練位置都早於該輪驗證位置，第二輪擴張歷史訓練範圍；不能提前讓第 4、5 列參與第一輪的 fit。

</details>

### 11. SciPy 的「至少」端點

```python
from scipy import stats
dist = stats.binom(n=4, p=0.5)
print(round(float(dist.pmf(3)), 4))
print(round(float(1 - dist.cdf(2)), 4))
print(round(float(1 - dist.cdf(3)), 4))
```

<details><summary>答案與理由</summary>

```text
0.25
0.3125
0.0625
```

依序為恰好 3 次、至少 3 次、超過 3 次；CDF 包含傳入的端點。

</details>

### 12. 共用預設串列與淺複製

```python
def mark(value, labels=[]):
    labels.append(value)
    return labels
print(mark('A'))
print(mark('B'))
rows = [[1], [2]]
backup = rows.copy()
rows[0].append(9)
rows[1] = [0]
print(backup)
```

<details><summary>答案與理由</summary>

```text
['A']
['A', 'B']
[[1, 9], [2]]
```

預設 list 在函式定義時建立一次；需要每次新建可改用 `labels=None`，函式內再建立 list。`copy()` 只複製外層；內層 append 改共用物件，替換 `rows[1]` 則只改 rows 的外層參照。獨立備份巢狀資料可用 `copy.deepcopy`。

</details>

## 七、錯題回查與來源

| 卡住的考點 | 回查題目 |
|---|---|
| 基本篩選、分組、合併、Int64、缺失、fit、weighted F1 | [第四回第 41～50 題](../regular-level/L22_第四回_真題難度校準模擬考/第四回_L22_題本_50題.md#第-41-題) |
| Pipeline、可變預設參數、分批讀取、過去 rolling、weighted F1 | [第五回第 43～48 題](../regular-level/L22_第五回_情境整合模擬考/第五回_L22_題本_50題.md#第-43-題) |
| transform、排序去重、交叉驗證洩漏 | [第六回第 8 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-8-題)、[第 11 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-11-題)、[第 17 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-17-題) |
| 分批平均、混淆矩陣、數值清理、淺複製 | [第六回第 21 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-21-題)、[第 26 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-26-題)、[第 30 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-30-題)、[第 37 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-37-題) |
| Welch t、merge 基數、StandardScaler 手算 | [第六回第 40 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-40-題)、[第 43 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-43-題)、[第 47 題](../regular-level/L22_第六回_擬真混合模擬考/第六回_L22_題本_50題.md#第-47-題) |

公式或工具用途仍不熟，回 [考前速查表](./02_考前速查表.md)；能讀輸出後，再用 [主動回想題](./03_主動回想題.md) 練習說出判斷理由。自測錯題隔 1、3、7 天重答，每次都指出「最容易誤讀的資料範圍或分母」。

API 核對：[Pandas groupby](https://pandas.pydata.org/docs/user_guide/groupby.html)、[merge](https://pandas.pydata.org/docs/reference/api/pandas.merge.html)、[scikit-learn 洩漏與 Pipeline](https://scikit-learn.org/stable/common_pitfalls.html)、[混淆矩陣](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.confusion_matrix.html)、[SciPy binom](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.binom.html)、[Seaborn barplot](https://seaborn.pydata.org/generated/seaborn.barplot.html)、[Python 預設參數](https://docs.python.org/3/tutorial/controlflow.html#default-argument-values)。

本頁所有 Python 區塊均以獨立命名空間執行並比對列示輸出。核對環境：Python 3.12.14、pandas 2.2.3、scikit-learn 1.8.0、SciPy 1.17.0、Seaborn 0.13.2；Seaborn 範例另以非互動後端檢查長條高度。
