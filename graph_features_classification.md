# 🎓 Neo4j: Klasifikasi Mahasiswa dengan Fitur Algoritma Graf
## Centrality & Community Detection (tanpa Embedding) — Mahasiswa · Mata Kuliah · Lab Riset

> **Prasyarat**: Neo4j 5.x + GDS 2.5+ aktif. Gunakan database kosong (atau jalankan *Reset* di bagian 1).
> **Ide utama**: fitur ML tidak berasal dari `FastRP`, tetapi dari **ukuran struktural yang bisa dijelaskan**: degree, PageRank, betweenness, closeness, triangle count, clustering coefficient, dan komunitas (Louvain). Pendekatan ini mengikuti alur artikel Neo4j *"Using Neo4j Graph Data Science to improve machine learning models"*: **hitung fitur graf → bandingkan model baseline vs model + fitur graf**.

---

## 📋 Daftar Isi
1. [Data Sintetis (Mahasiswa, MK, Lab)](#1-data-sintetis)
2. [Relasi AMBIL, GABUNG, KENAL](#2-relasi)
3. [Label Target](#3-label-target)
4. [Fitur Graf: Centrality & Community](#4-fitur-graf)
5. [Analisis Fitur (Sebelum Training)](#5-analisis-fitur)
6. [Pipeline Klasifikasi: Baseline vs Fitur Graf](#6-pipeline-klasifikasi)
7. [Prediksi & Evaluasi](#7-prediksi--evaluasi)
8. [Opsional: Versi Python (graphdatascience + scikit-learn)](#8-opsional-python)
9. [Catatan, Jebakan, dan Troubleshooting](#9-catatan)

---

## Mengapa data dibuat ulang?

Pada contoh embedding sebelumnya, rata-rata nilai hampir selalu `3.5`, sehingga ~87% mahasiswa berlabel `1`, dan graf `KENAL` hanya berupa 5 *clique* identik (semua node punya degree sama). Dengan struktur seperti itu centrality **tidak informatif**.

Versi ini membuat data yang tetap deterministik (tanpa `rand()`, jadi hasilnya sama di setiap run) tetapi:

| Aspek | Versi lama | Versi ini |
|---|---|---|
| Jumlah mahasiswa | 30 | **150** (cukup untuk split train/test) |
| Ukuran Lab | seragam | **tidak seragam** (30 / 40 / 10 / 40 / 30) |
| Jumlah MK per mahasiswa | 2–4 (pola tetap) | 2–5 (bervariasi) |
| Struktur `KENAL` | 5 clique identik | graf **non-clique**, degree bervariasi |
| Distribusi label | ~87% kelas 1 | **±47% kelas 1** (seimbang) |

> ℹ️ Data ini **sintetis**. Nilai sengaja dibuat berkorelasi dengan lab dan struktur pertemanan agar demo bermakna. Pada data nyata, besarnya peningkatan akurasi akan berbeda.

---

## 1. Data Sintetis

### 🔹 Reset (opsional, hati-hati: menghapus semua data di database ini)
```cypher
MATCH (n) DETACH DELETE n;
CALL gds.graph.drop('kenalGraph', false) YIELD graphName;
CALL gds.graph.drop('akademikGraph', false) YIELD graphName;
CALL gds.graph.drop('Pipeline_Graph', false) YIELD graphName;
```

### 🔹 5 Mata Kuliah
```cypher
CREATE
  (:MataKuliah {id: 'ETI',  nama: 'Etika Teknologi Informasi'}),
  (:MataKuliah {id: 'GRAF', nama: 'Graf Pengetahuan'}),
  (:MataKuliah {id: 'PBA',  nama: 'Pengolahan Bahasa Alami'}),
  (:MataKuliah {id: 'SE',   nama: 'Sistem Enterprise'}),
  (:MataKuliah {id: 'RO',   nama: 'Riset Operasional'});
```

### 🔹 5 Lab Riset
```cypher
CREATE
  (:LabRiset {id: 'ADDI', nama: 'ADDI Lab'}),
  (:LabRiset {id: 'IKTI', nama: 'IKTI Lab'}),
  (:LabRiset {id: 'MSI',  nama: 'MSI Lab'}),
  (:LabRiset {id: 'SE',   nama: 'SE Lab'}),
  (:LabRiset {id: 'RDIB', nama: 'RDIB Lab'});
```

### 🔹 150 Mahasiswa
```cypher
UNWIND range(1, 150) AS i
CREATE (:Mahasiswa {id: 'NRP' + toString(i), nama: 'Mahasiswa ' + toString(i)});

CREATE CONSTRAINT mhs_id IF NOT EXISTS FOR (m:Mahasiswa) REQUIRE m.id IS UNIQUE;
```

---

## 2. Relasi

### 🔹 `GABUNG` (Mahasiswa → LabRiset)
Lab dipilih dengan rumus `(i*i + i/3) % 5` (pembagian integer), sehingga ukuran lab tidak seragam.
```cypher
UNWIND range(1, 150) AS i
WITH i, ['ADDI','IKTI','MSI','SE','RDIB'][(i*i + i/3) % 5] AS labId
MATCH (m:Mahasiswa {id: 'NRP' + toString(i)}), (l:LabRiset {id: labId})
MERGE (m)-[:GABUNG {tahun_masuk: 2025}]->(l);
```

### 🔹 `AMBIL` (Mahasiswa → MataKuliah) + `nilai`
- Jumlah MK: `2 + (i*3 + i/7) % 4` → **2 sampai 5**
- MK dimulai dari indeks `(i*i + i/5) % 5`, lalu berurutan (melingkar)
- `nilai` (skala 1–4) dipengaruhi lab + variasi kecil, dan **tidak dipakai sebagai fitur** (lihat bagian 9)

```cypher
UNWIND range(1, 150) AS i
WITH i,
     (i*i + i/3) % 5           AS labIdx,
     2 + (i*3 + i/7) % 4       AS jumlah_mk,
     (i*i + i/5) % 5           AS start
UNWIND range(0, jumlah_mk - 1) AS k
WITH i, k,
     ['ETI','GRAF','PBA','SE','RO'][(start + k) % 5] AS mkId,
     [0.75, 0.25, 0.5, -0.25, -0.5][labIdx]          AS offset
MATCH (m:Mahasiswa {id: 'NRP' + toString(i)}), (mk:MataKuliah {id: mkId})
MERGE (m)-[r:AMBIL]->(mk)
SET r.nilai = 2.75 + offset + (((i*3 + k*5 + i/4) % 5) - 2) * 0.25;
```

### 🔹 `KENAL` (antar Mahasiswa)
Dua mahasiswa "saling kenal" jika **satu lab** dan **minimal satu MK yang sama**. Bobot = jumlah MK yang sama + 1 (lab).
Relasi disimpan dengan satu arah, lalu diperlakukan **UNDIRECTED** saat proyeksi.
```cypher
MATCH (m1:Mahasiswa)-[:GABUNG]->(:LabRiset)<-[:GABUNG]-(m2:Mahasiswa)
WHERE elementId(m1) < elementId(m2)
MATCH (m1)-[:AMBIL]->(mk:MataKuliah)<-[:AMBIL]-(m2)
WITH m1, m2, count(DISTINCT mk) AS sharedCourses
MERGE (m1)-[r:KENAL]->(m2)
SET r.sharekenal = sharedCourses + 1;
```

### 🔹 Verifikasi
```cypher
MATCH (n) RETURN labels(n)[0] AS node, count(*) AS jumlah
UNION ALL
MATCH ()-[r]->() RETURN type(r) AS node, count(*) AS jumlah;
```

---

## 3. Label Target

`target_label = 1` (High Performer) jika rata-rata nilai ≥ 3.0, selain itu `0`.
```cypher
MATCH (m:Mahasiswa)-[a:AMBIL]->()
WITH m, avg(a.nilai) AS avg_nilai
SET m.avg_nilai = avg_nilai,   // hanya untuk analisis, JANGAN dipakai sebagai fitur
    m.target_label = CASE WHEN avg_nilai >= 3.0 THEN 1 ELSE 0 END;
```

```cypher
MATCH (m:Mahasiswa)
RETURN m.target_label AS label, count(*) AS jumlah
ORDER BY label;
```
✅ Rasio kelas seharusnya mendekati seimbang (±53% : ±47%).

---

## 4. Fitur Graf

Ada dua "sudut pandang" graf:

| Proyeksi | Isi | Pertanyaan yang dijawab |
|---|---|---|
| `kenalGraph` | Mahasiswa — `KENAL` | Seberapa penting/terhubung mahasiswa dalam jejaring sosialnya? |
| `akademikGraph` | Mahasiswa, MK, Lab — `AMBIL`, `GABUNG` | Seberapa sentral mahasiswa dalam struktur akademik? Seberapa populer MK/lab yang ia ikuti? |

### 4.1 Proyeksi `kenalGraph`
```cypher
CALL gds.graph.project(
  'kenalGraph',
  'Mahasiswa',
  {KENAL: {orientation: 'UNDIRECTED', properties: 'sharekenal'}}
) YIELD graphName, nodeCount, relationshipCount;
```

### 4.2 Centrality pada `kenalGraph`
```cypher
// Degree (jumlah teman) dan weighted degree (total kekuatan pertemanan)
CALL gds.degree.write('kenalGraph', {writeProperty: 'deg_kenal'})
YIELD nodePropertiesWritten;

CALL gds.degree.write('kenalGraph', {
  writeProperty: 'wdeg_kenal',
  relationshipWeightProperty: 'sharekenal'
}) YIELD nodePropertiesWritten;

// PageRank (berbobot): pengaruh berdasarkan kualitas teman
CALL gds.pageRank.write('kenalGraph', {
  writeProperty: 'pr_kenal',
  relationshipWeightProperty: 'sharekenal',
  dampingFactor: 0.85,
  maxIterations: 50
}) YIELD nodePropertiesWritten, ranIterations;

// Betweenness: seberapa sering menjadi "jembatan"
CALL gds.betweenness.write('kenalGraph', {writeProperty: 'btw_kenal'})
YIELD nodePropertiesWritten;

// Closeness: kedekatan rata-rata ke semua mahasiswa lain
CALL gds.closeness.write('kenalGraph', {
  writeProperty: 'clo_kenal',
  useWassermanFaust: true
}) YIELD nodePropertiesWritten;
```

### 4.3 Struktur lokal (triangle & clustering)
```cypher
CALL gds.triangleCount.write('kenalGraph', {writeProperty: 'tri_kenal'})
YIELD nodeCount, globalTriangleCount;

CALL gds.localClusteringCoefficient.write('kenalGraph', {writeProperty: 'lcc_kenal'})
YIELD nodeCount, averageClusteringCoefficient;
```

### 4.4 Community Detection (Louvain)
```cypher
CALL gds.louvain.write('kenalGraph', {
  writeProperty: 'komunitas_kenal',
  relationshipWeightProperty: 'sharekenal'
}) YIELD communityCount, modularity;
```

> ⚠️ `komunitas_kenal` adalah **ID nominal** (nomor 5 bukan "lebih besar" dari nomor 2). Jangan dimasukkan langsung sebagai fitur numerik ke Logistic Regression. Kita ubah menjadi fitur numerik yang bermakna: **ukuran komunitas**.

```cypher
MATCH (m:Mahasiswa)
WITH m.komunitas_kenal AS c, collect(m) AS anggota
UNWIND anggota AS m
SET m.ukuran_komunitas = size(anggota);
```

**Cek: apakah komunitas Louvain ≈ Lab?** (wajar, karena `KENAL` hanya terbentuk di dalam lab yang sama)
```cypher
MATCH (m:Mahasiswa)-[:GABUNG]->(l:LabRiset)
RETURN m.komunitas_kenal AS komunitas, l.id AS lab, count(*) AS jumlah
ORDER BY komunitas, jumlah DESC;
```

### 4.5 Proyeksi `akademikGraph` (heterogen)
`nilai` **tidak** diproyeksikan, agar tidak ada kebocoran label.
```cypher
CALL gds.graph.project(
  'akademikGraph',
  ['Mahasiswa', 'MataKuliah', 'LabRiset'],
  {
    AMBIL:   {orientation: 'UNDIRECTED'},
    GABUNG:  {orientation: 'UNDIRECTED'}
  }
) YIELD graphName, nodeCount, relationshipCount;
```

```cypher
// PageRank & Betweenness pada graf heterogen (ditulis ke Mahasiswa, MK, dan Lab)
CALL gds.pageRank.write('akademikGraph', {writeProperty: 'pr_akademik', maxIterations: 50})
YIELD nodePropertiesWritten;

CALL gds.betweenness.write('akademikGraph', {writeProperty: 'btw_akademik'})
YIELD nodePropertiesWritten;

// Degree per jenis relasi:
//  - deg_ambil  -> pada Mahasiswa = jumlah MK diambil (BASELINE), pada MK = jumlah peserta
//  - deg_gabung -> pada Lab = jumlah anggota lab
CALL gds.degree.write('akademikGraph', {
  relationshipTypes: ['AMBIL'], writeProperty: 'deg_ambil'
}) YIELD nodePropertiesWritten;

CALL gds.degree.write('akademikGraph', {
  relationshipTypes: ['GABUNG'], writeProperty: 'deg_gabung'
}) YIELD nodePropertiesWritten;
```

### 4.6 Agregasi ke level mahasiswa
Skor milik MK dan Lab "diturunkan" ke mahasiswa, dan skor teman "diagregasi" (*neighborhood aggregation*).
```cypher
// (a) Rata-rata popularitas MK yang diambil
MATCH (m:Mahasiswa)-[:AMBIL]->(mk:MataKuliah)
WITH m, avg(mk.deg_ambil) AS v
SET m.avg_mk_popularitas = v;

// (b) Ukuran lab tempat mahasiswa bergabung
MATCH (m:Mahasiswa)-[:GABUNG]->(l:LabRiset)
SET m.ukuran_lab = l.deg_gabung;

// (c) Rata-rata PageRank teman (pengaruh lingkungan sosial)
MATCH (m:Mahasiswa)
OPTIONAL MATCH (m)-[:KENAL]-(t:Mahasiswa)
WITH m, avg(t.pr_kenal) AS v
SET m.avg_pr_teman = coalesce(v, 0.0);
```

### 4.7 Ringkasan fitur

| Fitur | Sumber | Makna |
|---|---|---|
| `deg_ambil` | Degree (AMBIL) | **Baseline**: jumlah MK diambil |
| `deg_kenal` | Degree | Jumlah teman |
| `wdeg_kenal` | Weighted degree | Kekuatan total pertemanan |
| `pr_kenal` | PageRank | Pengaruh di jejaring sosial |
| `btw_kenal` | Betweenness | Peran "jembatan" antar-kelompok |
| `clo_kenal` | Closeness | Kedekatan ke seluruh jejaring |
| `tri_kenal` | Triangle count | Jumlah segitiga pertemanan |
| `lcc_kenal` | Clustering coefficient | Seberapa rapat teman-temannya saling kenal |
| `ukuran_komunitas` | Louvain | Ukuran komunitas tempat berada |
| `pr_akademik` | PageRank (heterogen) | Sentralitas di struktur mhs–MK–lab |
| `btw_akademik` | Betweenness (heterogen) | Peran jembatan di struktur akademik |
| `avg_mk_popularitas` | Agregasi MK | Seberapa "ramai" MK yang diambil |
| `ukuran_lab` | Agregasi Lab | Besar lab tempat bergabung |
| `avg_pr_teman` | Agregasi tetangga | Kualitas/pengaruh lingkaran pertemanan |

---

## 5. Analisis Fitur

### 🔹 Korelasi fitur dengan label
```cypher
MATCH (m:Mahasiswa)
WITH collect(m) AS ms
UNWIND ['deg_ambil','deg_kenal','wdeg_kenal','pr_kenal','btw_kenal','clo_kenal',
        'tri_kenal','lcc_kenal','ukuran_komunitas','pr_akademik','btw_akademik',
        'avg_mk_popularitas','ukuran_lab','avg_pr_teman'] AS f
RETURN f AS fitur,
       round(gds.similarity.pearson(
         [x IN ms | toFloat(x.target_label)],
         [x IN ms | toFloat(x[f])]), 3) AS korelasi_dgn_label
ORDER BY abs(korelasi_dgn_label) DESC;
```
Nilai `null`/`NaN` berarti fitur konstan (tidak informatif).

### 🔹 Rasio High Performer per komunitas
```cypher
MATCH (m:Mahasiswa)
RETURN m.komunitas_kenal AS komunitas,
       count(*) AS anggota,
       round(avg(toFloat(m.target_label)), 2) AS rasio_high_performer
ORDER BY komunitas;
```

---

## 6. Pipeline Klasifikasi

Kita latih **dua pipeline** dengan split dan algoritma yang sama, hanya beda fitur:

- 🅰️ **Baseline**: hanya `deg_ambil` (jumlah MK — informasi tabular sederhana)
- 🅱️ **Fitur Graf**: 13 fitur centrality, community, dan agregasi

### 6.1 Proyeksi untuk Pipeline
Hanya node `Mahasiswa` (beserta fitur dan label) yang dibutuhkan.
```cypher
CALL gds.graph.project(
  'Pipeline_Graph',
  {Mahasiswa: {properties: [
      'target_label', 'deg_ambil',
      'deg_kenal','wdeg_kenal','pr_kenal','btw_kenal','clo_kenal',
      'tri_kenal','lcc_kenal','ukuran_komunitas',
      'pr_akademik','btw_akademik','avg_mk_popularitas','ukuran_lab','avg_pr_teman'
  ]}},
  {KENAL: {orientation: 'UNDIRECTED'}}
) YIELD graphName, nodeCount, relationshipCount;
```

### 6.2 Normalisasi fitur (MinMax) di dalam graf in-memory
Logistic Regression sensitif terhadap skala, jadi semua fitur diskalakan ke 0–1 dan digabung menjadi satu vektor.
```cypher
// Vektor fitur graf
CALL gds.scaleProperties.mutate('Pipeline_Graph', {
  nodeProperties: ['deg_kenal','wdeg_kenal','pr_kenal','btw_kenal','clo_kenal',
                   'tri_kenal','lcc_kenal','ukuran_komunitas',
                   'pr_akademik','btw_akademik','avg_mk_popularitas',
                   'ukuran_lab','avg_pr_teman'],
  scaler: 'MinMax',
  mutateProperty: 'fitur_graf'
}) YIELD nodePropertiesWritten;

// Vektor fitur baseline
CALL gds.scaleProperties.mutate('Pipeline_Graph', {
  nodeProperties: ['deg_ambil'],
  scaler: 'MinMax',
  mutateProperty: 'fitur_baseline'
}) YIELD nodePropertiesWritten;
```

### 6.3 Pipeline 🅱️ — Fitur Graf
```cypher
CALL gds.beta.pipeline.nodeClassification.create('PipeGraf') YIELD name;

// Kandidat model 1: Logistic Regression (penalty di-tuning otomatis)
CALL gds.beta.pipeline.nodeClassification.addLogisticRegression('PipeGraf', {
  penalty: {range: [0.001, 0.1]},
  maxEpochs: 200,
  tolerance: 0.001
}) YIELD name;

// Kandidat model 2: Random Forest (tahan terhadap skala & non-linearitas)
CALL gds.beta.pipeline.nodeClassification.addRandomForest('PipeGraf', {
  numberOfDecisionTrees: 50,
  maxDepth: 5
}) YIELD name;

CALL gds.beta.pipeline.nodeClassification.configureSplit('PipeGraf', {
  testFraction: 0.2,
  validationFolds: 5
}) YIELD splitConfig;

CALL gds.beta.pipeline.nodeClassification.selectFeatures('PipeGraf', ['fitur_graf'])
YIELD selectedFeatures;
```

### 6.4 Pipeline 🅰️ — Baseline
```cypher
CALL gds.beta.pipeline.nodeClassification.create('PipeBaseline') YIELD name;

CALL gds.beta.pipeline.nodeClassification.addLogisticRegression('PipeBaseline', {
  penalty: {range: [0.001, 0.1]}, maxEpochs: 200, tolerance: 0.001
}) YIELD name;

CALL gds.beta.pipeline.nodeClassification.addRandomForest('PipeBaseline', {
  numberOfDecisionTrees: 50, maxDepth: 5
}) YIELD name;

CALL gds.beta.pipeline.nodeClassification.configureSplit('PipeBaseline', {
  testFraction: 0.2, validationFolds: 5
}) YIELD splitConfig;

CALL gds.beta.pipeline.nodeClassification.selectFeatures('PipeBaseline', ['fitur_baseline'])
YIELD selectedFeatures;
```

### 6.5 Training
Gunakan `randomSeed` yang **sama** di kedua model agar perbandingan adil.
```cypher
// 🅱️ Fitur graf
CALL gds.beta.pipeline.nodeClassification.train('Pipeline_Graph', {
  pipeline: 'PipeGraf',
  targetNodeLabels: ['Mahasiswa'],
  modelName: 'Model_Graf',
  targetProperty: 'target_label',
  metrics: ['F1_WEIGHTED', 'ACCURACY'],   // metrik pertama dipakai untuk memilih model terbaik
  randomSeed: 42
}) YIELD modelInfo
RETURN 'Fitur Graf' AS model,
       modelInfo.bestParameters AS parameter_terbaik,
       modelInfo.metrics.ACCURACY.validation.avg AS acc_validasi,
       modelInfo.metrics.ACCURACY.test           AS acc_test,
       modelInfo.metrics.F1_WEIGHTED.test        AS f1_test;
```
```cypher
// 🅰️ Baseline
CALL gds.beta.pipeline.nodeClassification.train('Pipeline_Graph', {
  pipeline: 'PipeBaseline',
  targetNodeLabels: ['Mahasiswa'],
  modelName: 'Model_Baseline',
  targetProperty: 'target_label',
  metrics: ['F1_WEIGHTED', 'ACCURACY'],
  randomSeed: 42
}) YIELD modelInfo
RETURN 'Baseline' AS model,
       modelInfo.bestParameters AS parameter_terbaik,
       modelInfo.metrics.ACCURACY.validation.avg AS acc_validasi,
       modelInfo.metrics.ACCURACY.test           AS acc_test,
       modelInfo.metrics.F1_WEIGHTED.test        AS f1_test;
```

#### 📊 Ekspektasi hasil
Pada simulasi saya terhadap data yang sama (Random Forest, 5-fold CV di Python), baseline hanya sedikit di atas tebakan acak (akurasi ≈ 0.53), sedangkan model dengan fitur graf mencapai ≈ 0.77. Angka dari GDS akan sedikit berbeda (implementasi centrality, split, dan seed berbeda), tetapi **arah perbedaannya** seharusnya sama. Dengan hanya ±30 node di test set, selisih beberapa persen bisa murni noise, jadi bandingkan juga `validation.avg` (hasil cross-validation).

---

## 7. Prediksi & Evaluasi

### 7.1 Tulis prediksi ke database
```cypher
CALL gds.beta.pipeline.nodeClassification.predict.write('Pipeline_Graph', {
  modelName: 'Model_Graf',
  targetNodeLabels: ['Mahasiswa'],
  writeProperty: 'prediksi_graf',
  predictedProbabilityProperty: 'prob_graf'      // list [P(kelas 0), P(kelas 1)]
}) YIELD nodePropertiesWritten;

CALL gds.beta.pipeline.nodeClassification.predict.write('Pipeline_Graph', {
  modelName: 'Model_Baseline',
  targetNodeLabels: ['Mahasiswa'],
  writeProperty: 'prediksi_baseline',
  predictedProbabilityProperty: 'prob_baseline'
}) YIELD nodePropertiesWritten;
```

### 7.2 Confusion matrix & metrik (graf vs baseline)
> ⚠️ Query ini memprediksi **semua** node, termasuk node yang dipakai saat training, sehingga hasilnya **optimistis (in-sample)**. Untuk klaim performa, gunakan `acc_test` / `f1_test` dari bagian 6.5. Query ini berguna untuk melihat pola kesalahan.

```cypher
MATCH (m:Mahasiswa)
UNWIND [
  {model: 'Fitur Graf', pred: m.prediksi_graf},
  {model: 'Baseline',   pred: m.prediksi_baseline}
] AS r
WITH r.model AS model,
     sum(CASE WHEN m.target_label = 1 AND r.pred = 1 THEN 1 ELSE 0 END) AS TP,
     sum(CASE WHEN m.target_label = 0 AND r.pred = 0 THEN 1 ELSE 0 END) AS TN,
     sum(CASE WHEN m.target_label = 0 AND r.pred = 1 THEN 1 ELSE 0 END) AS FP,
     sum(CASE WHEN m.target_label = 1 AND r.pred = 0 THEN 1 ELSE 0 END) AS FN
RETURN model, TP, TN, FP, FN,
       round(toFloat(TP + TN) / (TP + TN + FP + FN), 4) AS accuracy,
       CASE WHEN TP + FP = 0 THEN null ELSE round(toFloat(TP) / (TP + FP), 4) END AS precision,
       CASE WHEN TP + FN = 0 THEN null ELSE round(toFloat(TP) / (TP + FN), 4) END AS recall;
```

### 7.3 Mahasiswa dengan peluang High Performer tertinggi
```cypher
MATCH (m:Mahasiswa)
RETURN m.id AS NRP, m.target_label AS aktual, m.prediksi_graf AS prediksi,
       round(m.prob_graf[1], 3) AS p_high_performer,
       m.deg_kenal AS teman, round(m.pr_kenal, 4) AS pagerank
ORDER BY p_high_performer DESC
LIMIT 10;
```

### 7.4 Salah prediksi: apa kata strukturnya?
```cypher
MATCH (m:Mahasiswa)
WHERE m.prediksi_graf <> m.target_label
RETURN m.id AS NRP, m.target_label AS aktual, m.prediksi_graf AS prediksi,
       m.deg_kenal AS teman, m.ukuran_komunitas AS ukuran_komunitas,
       round(m.avg_nilai, 2) AS avg_nilai
ORDER BY abs(m.avg_nilai - 3.0) ASC
LIMIT 15;
```
Kesalahan biasanya terjadi pada mahasiswa dengan `avg_nilai` mendekati batas 3.0, yang memang sulit dipisahkan.

---

## 8. Opsional: Python

Cara ini mirip artikel Medium: ambil fitur graf dari Neo4j, lalu bandingkan model scikit-learn **dengan dan tanpa** fitur graf. Berguna untuk melihat *feature importance*.

```bash
pip install graphdatascience scikit-learn pandas
```

```python
import pandas as pd
from graphdatascience import GraphDataScience
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_validate, StratifiedKFold

gds = GraphDataScience("bolt://localhost:7687", auth=("neo4j", "password"))

BASELINE = ["deg_ambil"]
GRAF = ["deg_kenal", "wdeg_kenal", "pr_kenal", "btw_kenal", "clo_kenal",
        "tri_kenal", "lcc_kenal", "ukuran_komunitas", "pr_akademik",
        "btw_akademik", "avg_mk_popularitas", "ukuran_lab", "avg_pr_teman"]

df = gds.run_cypher(f"""
    MATCH (m:Mahasiswa)
    RETURN m.id AS id, m.target_label AS y,
           {", ".join(f"m.{c} AS {c}" for c in BASELINE + GRAF)}
""")

y = df["y"]
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
rf = RandomForestClassifier(n_estimators=200, max_depth=5, random_state=42)

for nama, fitur in [("Baseline", BASELINE), ("Baseline + Graf", BASELINE + GRAF)]:
    s = cross_validate(rf, df[fitur], y, cv=cv, scoring=["accuracy", "roc_auc", "f1"])
    print(f"{nama:16s} acc={s['test_accuracy'].mean():.3f} "
          f"auc={s['test_roc_auc'].mean():.3f} f1={s['test_f1'].mean():.3f}")

# Feature importance
rf.fit(df[BASELINE + GRAF], y)
print(pd.Series(rf.feature_importances_, index=BASELINE + GRAF)
        .sort_values(ascending=False).round(3))
```

---

## 9. Catatan

### ⚠️ Jebakan yang perlu dihindari

| Masalah | Penjelasan | Solusi |
|---|---|---|
| **Label leakage** | `target_label` dihitung dari `nilai`. Jika `nilai` dipakai sebagai bobot relasi atau fitur, model "mencontek". | Jangan proyeksikan `nilai`; `avg_nilai` hanya untuk analisis |
| **Komunitas ≈ Lab** | `KENAL` hanya terbentuk dalam satu lab, jadi Louvain hampir pasti mengulang struktur lab | Anggap komunitas sebagai fitur pelengkap; kekuatan utama ada di centrality/struktur lokal |
| **ID komunitas bukan angka bermakna** | `komunitas_kenal = 4` tidak "lebih besar" dari `2` | Pakai `ukuran_komunitas`, atau one-hot encoding di Python |
| **Fitur transduktif** | Fitur graf dihitung di seluruh graf (tanpa label), termasuk node test | Aman dari kebocoran label; untuk deployment ketat, hitung ulang fitur tiap kali ada data baru |
| **Multikolinearitas** | `deg_kenal`, `wdeg_kenal`, `tri_kenal` saling berkorelasi tinggi | Cek korelasi (bagian 5); Random Forest lebih toleran dibanding Logistic Regression |
| **Test set kecil** | 20% × 150 = ±30 node | Andalkan `validation.avg`, perbesar jumlah mahasiswa, atau ulangi dengan beberapa `randomSeed` |
| **Prediksi in-sample** | `predict.write` menimpa semua node | Evaluasi performa dari metrik `test`, bukan confusion matrix global |

### 🔧 Troubleshooting

| Error | Penyebab | Solusi |
|---|---|---|
| `Procedure ... not found: gds.beta.pipeline.nodeClassification.*` | Versi GDS berbeda | Cek `CALL gds.list()`; pada GDS lebih baru namanya bisa tanpa `.beta` |
| `Property 'xxx' not found in graph` | Fitur belum ditulis sebelum `gds.graph.project` | Jalankan bagian 4 dulu, lalu proyeksi ulang (`gds.graph.drop` → `project`) |
| `Graph with name ... already exists` | Proyeksi lama masih ada | `CALL gds.graph.drop('nama', false)` |
| `The graph is not undirected` (triangleCount) | Orientation tidak `UNDIRECTED` | Proyeksikan `KENAL` dengan `orientation: 'UNDIRECTED'` |
| `Model with name ... already exists` | Model dengan nama sama sudah tersimpan | `CALL gds.model.drop('Model_Graf', false)` |
| Metrik `NaN` pada korelasi | Fitur konstan | Hapus dari daftar fitur |

### 🧹 Cleanup
```cypher
CALL gds.graph.drop('kenalGraph', false)     YIELD graphName;
CALL gds.graph.drop('akademikGraph', false)  YIELD graphName;
CALL gds.graph.drop('Pipeline_Graph', false) YIELD graphName;

CALL gds.model.drop('Model_Graf', false)     YIELD modelInfo;
CALL gds.model.drop('Model_Baseline', false) YIELD modelInfo;

// Pipeline: gunakan gds.pipeline.drop('nama') (GDS 2.5+) atau gds.beta.pipeline.drop('nama') (versi lama)
CALL gds.pipeline.drop('PipeGraf')     YIELD pipelineName;
CALL gds.pipeline.drop('PipeBaseline') YIELD pipelineName;
```

### 💡 Ide Lanjutan
1. **Pipeline-native feature steps**: `addNodeProperty('PipeGraf', 'pageRank', {...})` agar fitur dihitung ulang otomatis saat `predict` pada graf baru.
2. Tambahkan **`gds.wcc`** (komponen terhubung) atau **`gds.labelPropagation`** sebagai pembanding Louvain.
3. Coba **`gds.betweenness`** dengan `samplingSize` jika graf diperbesar (ribuan node).
4. Ganti target: prediksi **lab** mahasiswa (multi-kelas) memakai fitur agregasi MK, atau prediksi relasi `KENAL` baru dengan *link prediction pipeline*.

---

*Jalankan blok Cypher secara berurutan di Neo4j Browser (satu pernyataan per eksekusi, atau gunakan `:param`/multi-statement dengan titik koma).*
