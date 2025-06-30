# パフォーマンス問題分析演習 - Observability研修教材

## 🎯 学習目標

この演習では、OpenTelemetryトレーシングデータを分析して：
1. **ボトルネックの特定**：実際のSpanデータから性能問題を発見
2. **根本原因分析**：なぜその処理が遅いのかを理解
3. **改善案の立案**：具体的な解決策を考える
4. **優先順位付け**：影響度の高い問題から解決する判断力

---

## 📊 実際のトレーシングデータ

### slow_tfidf_search の性能問題（実測値）

```
🔍 Span: search_api
   Duration: 1664.55 ms ← 全体の処理時間
   └── slow_tfidf_search
       Duration: 1664.30 ms
       ├── slow_preprocess_query: 213.65 ms
       ├── slow_vectorize_query: 555.02 ms  
       ├── slow_compute_similarity: 532.33 ms
       └── slow_process_results: 362.65 ms
           ├── slow_generate_snippet (×5): 60-95ms each
           └── inefficient_bubble_sort: 12.73 ms
```

**通常のTF-IDF検索**: 414ms  
**遅いTF-IDF検索**: 1664ms（**約4倍遅い**）

---

## 🔍 ボトルネック分析演習

### 問題1: 前処理の無駄な処理

**観測されたデータ**:
```json
{
  "span_name": "slow_preprocess_query",
  "duration": 213.65,
  "attributes": {
    "query.original": "blue whale",
    "query.processed": "blue whale", 
    "bottleneck.dummy_operations": 500000
  }
}
```

**🤔 分析課題**:
- なぜ「blue whale」という短いクエリの前処理に213msもかかるのか？
- `dummy_operations: 500000` は何を意味するか？
- この処理は本当に必要か？

**💡 改善案**:
```python
# ❌ 問題のあるコード
for i in range(50000):  # 不要なループ
    temp_string = processed_query.upper().lower().strip()
time.sleep(0.2)  # 不要な待機

# ✅ 改善後
processed_query = preprocess_text(query)  # 1回だけ実行
# sleepを削除
```

**期待される改善**: 213ms → 1ms（**99.5%短縮**）

---

### 問題2: 重複したベクトル化処理

**観測されたデータ**:
```json
{
  "span_name": "slow_vectorize_query", 
  "duration": 555.02,
  "attributes": {
    "vector.shape": "(1, 5000)",
    "bottleneck.duplicate_vectorizations": 10
  }
}
```

**🤔 分析課題**:
- なぜ同じクエリを10回ベクトル化するのか？
- 1回の処理時間は約55ms、残りの500msは何？
- メモリ使用量への影響は？

**💡 改善案**:
```python
# ❌ 問題のあるコード  
query_vector = tfidf_vectorizer.transform([processed_query])
for i in range(10):  # 不要な重複処理
    duplicate_vector = tfidf_vectorizer.transform([processed_query])
    time.sleep(0.05)

# ✅ 改善後
query_vector = tfidf_vectorizer.transform([processed_query])
# 重複処理とsleepを削除
```

**期待される改善**: 555ms → 55ms（**90%短縮**）

---

### 問題3: 類似度計算の無駄な再実行

**観測されたデータ**:
```json
{
  "span_name": "slow_compute_similarity",
  "duration": 532.33, 
  "attributes": {
    "similarity.matrix_size": 18,
    "bottleneck.recalculation_operations": 90
  }
}
```

**🤔 分析課題**:
- 18文書の類似度計算がなぜ532msもかかるのか？
- `recalculation_operations: 90` の意味は？
- 正常な処理時間はどの程度か？

**💡 改善案**:
```python
# ❌ 問題のあるコード
similarities = cosine_similarity(query_vector, tfidf_matrix)
for i in range(5):  # 5回も再計算
    temp_similarities = cosine_similarity(query_vector, tfidf_matrix)
    time.sleep(0.1)

# ✅ 改善後  
similarities = cosine_similarity(query_vector, tfidf_matrix)
# 再計算とsleepを削除
```

**期待される改善**: 532ms → 32ms（**94%短縮**）

---

### 問題4: スニペット生成の非効率性

**観測されたデータ**:
```json
{
  "span_name": "slow_generate_snippet",
  "duration": 95.63,
  "attributes": {
    "book.id": "melville-moby_dick.txt"
  }
}
```

**🤔 分析課題**:
- 通常のスニペット生成は20-30ms、なぜ95msもかかるのか？
- 各スニペット生成の後に何が起きているか？
- 並列処理は可能か？

**💡 改善案**:
```python
# ❌ 問題のあるコード
with tracer.start_as_current_span("slow_generate_snippet"):
    snippet = get_snippet(book_info['raw_text'], query)
    time.sleep(0.03)  # 不要な遅延

# ✅ 改善後
with tracer.start_as_current_span("generate_snippet"):
    snippet = get_snippet(book_info['raw_text'], query)
    # sleepを削除

# 🚀 さらなる改善（並列処理）
import asyncio
snippets = await asyncio.gather(*[
    get_snippet_async(book['raw_text'], query) 
    for book in matching_books
])
```

**期待される改善**: 95ms×5 → 25ms×5（**並列実行で75ms**）

---

### 問題5: 非効率なソートアルゴリズム  

**観測されたデータ**:
```json
{
  "span_name": "inefficient_bubble_sort",
  "duration": 12.73,
  "attributes": {
    "bottleneck.bubble_sort_comparisons": 10
  }
}
```

**🤔 分析課題**:
- なぜバブルソートを使用するのか？
- 10要素で12.73msは適切か？
- Pythonの標準ソートとの比較は？

**💡 改善案**:
```python
# ❌ 問題のあるコード（バブルソート）
for i in range(min(n, 50)):
    for j in range(min(n-i-1, 50)):
        if results[j]['score'] < results[j+1]['score']:
            results[j], results[j+1] = results[j+1], results[j]
        time.sleep(0.001)

# ✅ 改善後（Pythonの組み込みソート）
results.sort(key=lambda x: x['score'], reverse=True)
```

**期待される改善**: 12.73ms → 0.1ms（**99%短縮**）

---

## 🎯 改善効果の試算

### 個別改善効果

| ボトルネック | 現在の時間 | 改善後 | 短縮率 |
|-------------|-----------|---------|--------|
| 前処理 | 213.65ms | 1ms | 99.5% |
| ベクトル化 | 555.02ms | 55ms | 90.1% |
| 類似度計算 | 532.33ms | 32ms | 94.0% |
| スニペット生成 | 475ms | 75ms | 84.2% |
| ソート | 12.73ms | 0.1ms | 99.2% |

### 総合改善効果

**改善前**: 1664ms  
**改善後**: 163ms（**90.2%短縮**）

**目標達成**: 通常のTF-IDF（414ms）より**60%高速化**

---

## 🏆 実践演習問題

### レベル1: 基本分析
1. トレーシングデータから最大のボトルネックを特定せよ
2. 各Spanの実行時間を分析し、問題箇所をランク付けせよ
3. `bottleneck.*` 属性から何が起きているかを推測せよ

### レベル2: 根本原因分析  
4. なぜ `slow_vectorize_query` が555msもかかるのか説明せよ
5. `recalculation_operations: 90` の計算根拠を示せ
6. 各ボトルネックが他の処理に与える影響を分析せよ

### レベル3: 改善提案
7. 最小の工数で最大の効果を得る改善案を3つ提案せよ
8. 並列処理による改善可能性を検討せよ  
9. 改善後の予想実行時間を計算せよ

### レベル4: アーキテクチャ設計
10. キャッシュ戦略を提案し、効果を試算せよ
11. マイクロサービス分割による改善案を検討せよ
12. リアルタイム監視で検知すべきメトリクスを定義せよ

---

## 💡 学習のポイント

### Observability分析の要点
1. **Span階層の理解**: 親子関係から処理フローを把握
2. **Duration分析**: 絶対時間と相対時間での比較
3. **属性活用**: カスタム属性から詳細情報を読み取り
4. **パターン認識**: 繰り返し処理や異常値の発見

### パフォーマンス改善の原則
1. **測定→分析→改善→検証** のサイクル
2. **80-20の法則**: 20%の改善で80%の効果
3. **早期最適化の回避**: 本当のボトルネックを特定してから
4. **トレードオフの考慮**: 速度vs精度、メモリvsCPU

### 実務への応用
- 本番環境での継続的な性能監視
- アラート閾値の適切な設定
- 段階的な改善とA/Bテスト
- チーム間でのナレッジ共有

---

## 📚 参考情報

### OpenTelemetry公式ドキュメント
- [Tracing API](https://opentelemetry.io/docs/instrumentation/python/manual/#traces)
- [Span Attributes](https://opentelemetry.io/docs/specs/otel/trace/semantic_conventions/)

### パフォーマンス分析ツール
- APMツール: Datadog, New Relic, Dynatrace
- プロファイラ: py-spy, cProfile, line_profiler
- 可視化: Jaeger, Zipkin, OpenTelemetry Collector

この演習を通じて、実際の本番環境でのパフォーマンス問題を特定・解決できるスキルを身に付けましょう！
