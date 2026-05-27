# 🎓 Neo4j Knowledge Graph & Machine Learning Pipeline
## Sistem Informasi Mahasiswa - Complete Cypher Workflow

> **Prasyarat**: Neo4j 5.x + GDS 2.5+ Plugin aktif  
> **Database**: Gunakan database kosong atau `CREATE DATABASE si_graph`

---

## 📋 Daftar Isi
1. [Schema & Data Initialization](#1-schema--data-initialization)
2. [Relationship Creation](#2-relationship-creation)
3. [Graph Projection & FastRP Embedding](#3-graph-projection--fastrp-embedding)
4. [Classification Label Preparation](#4-classification-label-preparation)
5. [Node Classification Pipeline](#5-node-classification-pipeline)
6. [Prediction & Evaluation](#6-prediction--evaluation)
7. [Troubleshooting & Notes](#7-troubleshooting--notes)

---

## 1. Schema & Data Initialization

### 🔹 Buat 5 Mata Kuliah
```cypher
CREATE 
  (mk1:MataKuliah {id: 'ETI', nama: 'Etika teknologi informasi'}),
  (mk2:MataKuliah {id: 'GRAF', nama: 'Graf Pengetahuan'}),
  (mk3:MataKuliah {id: 'PBA', nama: 'Pengolahan Bahasa Alami'}),
  (mk4:MataKuliah {id: 'SE', nama: 'Sistem Enterprise'}),
  (mk5:MataKuliah {id: 'RO', nama: 'Riset Operasional'});
```

### 🔹 Buat 5 Lab Riset
```cypher
CREATE 
  (lab1:LabRiset {id: 'ADDI', nama: 'ADDI Lab'}),
  (lab2:LabRiset {id: 'IKTI', nama: 'IKTI Lab'}),
  (lab3:LabRiset {id: 'MSI', nama: 'MSI Lab'}),
  (lab4:LabRiset {id: 'SE', nama: 'SE Lab'}),
  (lab5:LabRiset {id: 'RDIB', nama: 'RDIB Lab'});
```

### 🔹 Buat 30 Mahasiswa
```cypher
UNWIND range(1, 30) AS i
CREATE (m:Mahasiswa {id: 'NRP' + toString(i), nama: 'Mahasiswa ' + i});
```

---

## 2. Relationship Creation

### 🔹 Relasi `AMBIL` (Mahasiswa → MataKuliah)
**Aturan**: 
- ID habis bagi 5 → ambil **4 MK**
- ID habis bagi 3 → ambil **3 MK** 
- Lainnya → ambil **2 MK**
- Prioritas: 5 > 3 > 2

```cypher
UNWIND range(1, 30) AS i
WITH i, 'NRP' + toString(i) AS mId,
     CASE 
       WHEN i % 5 = 0 THEN 4
       WHEN i % 3 = 0 THEN 3
       ELSE 2
     END AS jumlah_mk,
     ['ETI','GRAF','PBA','SE','RO'] AS all_mk

UNWIND range(0, jumlah_mk - 1) AS idx
WITH mId, all_mk[(i + idx) % 5] AS mk_id, i, idx
MATCH (m:Mahasiswa {id: mId})
MATCH (mk:MataKuliah {id: mk_id})
MERGE (m)-[:AMBIL {nilai: 3.0 + ((i + idx) % 2)}]->(mk);
```

### 🔹 Relasi `GABUNG` (Mahasiswa → LabRiset)
Setiap mahasiswa bergabung ke **1 Lab** secara bergilir:
```cypher
UNWIND range(1, 30) AS i
WITH 'NRP' + toString(i) AS mId,
     ['ADDI','SE','MSI','RDIB','IKTI'][((i-1) % 5)] AS lab
MATCH (m:Mahasiswa {id: mId}), (l:LabRiset {id: lab})
MERGE (m)-[:GABUNG {tahun_masuk: 2025}]->(l);
```

### 🔹 Relasi `KENAL` (Mahasiswa ↔ Mahasiswa)
Dua mahasiswa "saling mengenal" jika berada di **Lab yang sama** DAN mengambil **MK yang sama**:
```cypher
MATCH (m1:Mahasiswa)-[:GABUNG]->(lab:LabRiset)<-[:GABUNG]-(m2:Mahasiswa),
      (m1)-[:AMBIL]->(mk:MataKuliah)<-[:AMBIL]-(m2)
WHERE elementId(m1) < elementId(m2)
WITH m1, m2, 
     count(DISTINCT lab) AS sharedLabs, 
     count(DISTINCT mk) AS sharedCourses
WHERE sharedLabs > 0 AND sharedCourses > 0
MERGE (m1)-[r:KENAL]-(m2)
SET r.sharekenal = sharedCourses + sharedLabs;
```

---

## 3. Graph Projection & FastRP Embedding

### 🔹 Graph Projection untuk FastRP
```cypher
// Drop jika sudah ada (aman dijalankan ulang)
CALL gds.graph.drop('MahasiswaGraph', {failIfMissing: true});

// Proyeksi graph dengan relationship properties
CALL gds.graph.project(
  'MahasiswaGraph',
  'Mahasiswa',
  {KENAL: {properties: 'sharekenal'}}
);
```

### 🔹 FastRP Embedding
```cypher
CALL gds.fastRP.write('MahasiswaGraph', {
  embeddingDimension: 8,
  iterationWeights: [0.5, 1.0, 0.5],
  writeProperty: 'emb',
  relationshipWeightProperty: 'sharekenal',
  randomSeed: 42
}) YIELD nodeCount;
```

### 🔹 Verifikasi Embedding
```cypher
MATCH (m:Mahasiswa) 
WHERE m.emb IS NOT NULL 
RETURN m.id, size(m.emb) AS dimensi_embedding 
LIMIT 3;
```

| Output yang Diharapkan |
|------------------------|
| `dimensi_embedding = 8` |
| `emb` bertipe `LIST<FLOAT>` |

---

## 4. Classification Label Preparation

### 🔹 Buat Target Label (`target_label`)
Label `1` = High Performer (rata-rata nilai ≥ 3.5), `0` = lainnya:
```cypher
MATCH (m:Mahasiswa)-[a:AMBIL]->()
WITH m, avg(a.nilai) AS avg_nilai
SET m.target_label = CASE WHEN avg_nilai >= 3.5 THEN 1 ELSE 0 END;
```

### 🔹 Cek Distribusi Kelas
```cypher
MATCH (m:Mahasiswa)
RETURN m.target_label AS labelmhs, count(m) AS Jumlah
ORDER BY labelmhs;
```

---

## 5. Node Classification Pipeline

### 🔹 Graph Projection untuk Pipeline ML
```cypher
CALL gds.graph.drop('Pipeline_Graph', {failIfMissing: true});

CALL gds.graph.project(
  'Pipeline_Graph',
  {Mahasiswa: {properties: ['emb', 'target_label']}},
  {KENAL: {properties: 'sharekenal'}}
);
```

### 🔹 Setup Pipeline Klasifikasi
```cypher
// 1. Buat Pipeline
CALL gds.beta.pipeline.nodeClassification.create('StudentPerfPipeline') YIELD name;

// 2. Tambahkan Logistic Regression dengan auto-tuning penalty
CALL gds.beta.pipeline.nodeClassification.addLogisticRegression('StudentPerfPipeline', {
  penalty: {range: [0.0, 0.1]},
  maxEpochs: 100,
  tolerance: 0.001
}) YIELD name;

// 3. Konfigurasi Train/Test Split & Cross-Validation
CALL gds.beta.pipeline.nodeClassification.configureSplit('StudentPerfPipeline', {
  testFraction: 0.2,        // 20% data di-holdout untuk testing
  validationFolds: 3        // 3-fold CV saat training
}) YIELD name;

// 4. Pilih fitur untuk klasifikasi (embedding FastRP)
CALL gds.beta.pipeline.nodeClassification.selectFeatures('StudentPerfPipeline', ['emb']) YIELD name;
```

### 🔹 Training Model
```cypher
CALL gds.beta.pipeline.nodeClassification.train('Pipeline_Graph', {
  pipeline: 'StudentPerfPipeline', 
  targetNodeLabels: ['Mahasiswa'],
  modelName: 'LogReg_StudentModel',
  targetProperty: 'target_label', 
  metrics: ['ACCURACY'],
  randomSeed: 42
}) YIELD modelInfo, trainMillis;
```

### 🔹 Simpan Prediksi ke Database
```cypher
CALL gds.beta.pipeline.nodeClassification.predict.write('Pipeline_Graph', {
  modelName: 'LogReg_StudentModel',
  writeProperty: 'prediksi_performa',
  predictedProbabilityProperty: 'probability'
}) YIELD nodePropertiesWritten, postProcessingMillis;
```

---

## 6. Prediction & Evaluation

### 🔹 Cek Hasil Prediksi per Mahasiswa
```cypher
MATCH (m:Mahasiswa)
WHERE m.prediksi_performa IS NOT NULL
RETURN 
  m.id AS NRP,
  m.target_label AS Label_Aktual,
  m.prediksi_performa AS Prediksi,
  m.probability[0] AS Prob_0,
  m.probability[1] AS Prob_1
ORDER BY m.probability[1] DESC
LIMIT 10;
```

### 🔹 Confusion Matrix & Metrik Evaluasi
```cypher
MATCH (m:Mahasiswa)
WHERE m.target_label IS NOT NULL 
  AND m.prediksi_performa IS NOT NULL
WITH 
  sum(CASE WHEN m.target_label = 1 AND m.prediksi_performa = 1 THEN 1 ELSE 0 END) AS TP,
  sum(CASE WHEN m.target_label = 0 AND m.prediksi_performa = 0 THEN 1 ELSE 0 END) AS TN,
  sum(CASE WHEN m.target_label = 0 AND m.prediksi_performa = 1 THEN 1 ELSE 0 END) AS FP,
  sum(CASE WHEN m.target_label = 1 AND m.prediksi_performa = 0 THEN 1 ELSE 0 END) AS FN
RETURN 
  {
    confusion_matrix: {
      actual_positive: {predicted_positive: TP, predicted_negative: FN},
      actual_negative: {predicted_positive: FP, predicted_negative: TN}
    },
    metrics: {
      accuracy: round(toFloat(TP + TN) / (TP + TN + FP + FN), 4),
      precision: CASE WHEN TP + FP = 0 THEN null ELSE round(toFloat(TP) / (TP + FP), 4) END,
      recall: CASE WHEN TP + FN = 0 THEN null ELSE round(toFloat(TP) / (TP + FN), 4) END,
      f1_score: CASE WHEN TP + FP = 0 OR TP + FN = 0 THEN null 
                ELSE round(2 * (toFloat(TP) / (TP + FP)) * (toFloat(TP) / (TP + FN)) / 
                          ((toFloat(TP) / (TP + FP)) + (toFloat(TP) / (TP + FN))), 4) 
                END
    }
  } AS hasil_evaluasi;
```

#### 📊 Output Contoh:
```json
{
  "confusion_matrix": {
    "actual_positive": {"predicted_positive": 8, "predicted_negative": 2},
    "actual_negative": {"predicted_positive": 1, "predicted_negative": 19}
  },
  "metrics": {
    "accuracy": 0.9000,
    "precision": 0.8889,
    "recall": 0.8000,
    "f1_score": 0.8421
  }
}
```

### 🔹 ROC Curve & AUC Calculation (Manual)
```cypher
// 1. Ekstrak probabilitas kelas positif (index 1)
MATCH (m:Mahasiswa)
WHERE m.target_label IS NOT NULL 
  AND m.probability IS NOT NULL
WITH collect({
  prob: m.probability[1], 
  actual: m.target_label
}) AS data

// 2. Generate threshold & hitung TPR/FPR
UNWIND range(0, 100) AS step
WITH data, toFloat(step) / 100 AS threshold
WITH data, threshold,
     toFloat(size([d IN data WHERE d.prob >= threshold AND d.actual = 1.0])) AS tp_count,
     toFloat(size([d IN data WHERE d.actual = 1.0])) AS actual_pos,
     toFloat(size([d IN data WHERE d.prob >= threshold AND d.actual = 0.0])) AS fp_count,
     toFloat(size([d IN data WHERE d.actual = 0.0])) AS actual_neg

// Hitung rate dengan aman (hindari division by zero)
WITH data, threshold,
     CASE WHEN actual_pos > 0 THEN tp_count / actual_pos ELSE 0.0 END AS tpr,
     CASE WHEN actual_neg > 0 THEN fp_count / actual_neg ELSE 0.0 END AS fpr

// ✅ ORDER BY harus SEBELUM collect()
ORDER BY threshold DESC
WITH collect({fpr: fpr, tpr: tpr, threshold: threshold}) AS roc_points

// 3. Hitung AUC dengan trapezoidal rule
UNWIND range(0, size(roc_points) - 2) AS i
WITH roc_points,
     roc_points[i].fpr AS x1, roc_points[i].tpr AS y1,
     roc_points[i+1].fpr AS x2, roc_points[i+1].tpr AS y2
WITH sum(
  CASE WHEN x2 > x1 THEN (x2 - x1) * (y1 + y2) / 2.0 ELSE 0 END
) AS auc_approx

RETURN round(auc_approx, 4) AS AUC_Score,
       CASE 
         WHEN auc_approx >= 0.9 THEN "Excellent"
         WHEN auc_approx >= 0.8 THEN "Good"
         WHEN auc_approx >= 0.7 THEN "Fair"
         WHEN auc_approx >= 0.6 THEN "Poor"
         ELSE "Fail"
       END AS Interpretasi;
```

#### 📈 Interpretasi AUC Score:
| AUC Score | Kualitas Model |
|-----------|---------------|
| ≥ 0.90 | 🏆 Excellent |
| 0.80–0.89 | ✅ Good |
| 0.70–0.79 | ⚠️ Fair |
| 0.60–0.69 | ❌ Poor |
| < 0.60 | 💥 Fail |

---

## 7. Troubleshooting & Notes

### ⚠️ Common Issues & Solutions

| Error / Masalah | Penyebab | Solusi |
|----------------|----------|--------|
| `Variable not defined` | Variabel tidak di-continue di `WITH` | Pastikan semua variabel yang dipakai di bawah ditulis eksplisit di `WITH` |
| `No such procedure apoc.*` | Plugin APOC belum aktif | Gunakan query native Cypher atau instal APOC via Neo4j Desktop → Plugins |
| `penalty must be numeric or map` | Format parameter salah di Pipeline | Gunakan `{range: [min, max]}` bukan array `[0.1, 0.01]` |
| `featureProperties must be numeric` | Embedding bukan `LIST<FLOAT>` | Cek: `RETURN m.emb IS LIST`, konversi jika perlu |
| `ORDER BY after aggregation` | Scoping error di Cypher | Pindahkan `ORDER BY` **sebelum** `WITH collect()` |

### 🧹 Cleanup (Opsional)
```cypher
// Drop graph projections setelah selesai
CALL gds.graph.drop('MahasiswaGraph', {failIfMissing: true});
CALL gds.graph.drop('Pipeline_Graph', {failIfMissing: true});

// Drop pipeline jika tidak dipakai lagi
CALL gds.beta.pipeline.nodeClassification.drop('StudentPerfPipeline');
```

### 💡 Best Practices
1. **Gunakan `elementId()`** bukan `id()` untuk uniqueness di Neo4j 5.x
2. **Selalu validasi tipe data** sebelum training: `RETURN m.emb IS LIST`
3. **Simpan `randomSeed`** untuk reproducibility hasil embedding & training
4. **Gunakan `write()`** bukan `mutate()` jika ingin properti tersimpan permanen di database
5. **Monitor class imbalance**: jika rasio `0:1` > 3:1, tambahkan `negativeClassWeight` di pipeline

---

> 🚀 **Next Steps**: 
> - Visualisasi hasil di **Neo4j Bloom** dengan visual rules berdasarkan `prediksi_performa`
> - Export embedding ke Python untuk analisis lanjutan: `CALL gds.graph.export.csv(...)`
> - Tambahkan algoritma lain di pipeline: `addRandomForest()`, `addMLP()`

*Document ini dapat dijalankan secara berurutan di Neo4j Browser. Pastikan GDS plugin aktif dengan `SHOW PLUGINS`.*
