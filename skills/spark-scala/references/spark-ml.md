# Spark MLlib — Pipeline API Reference

```
Transformer  — has transform(df): DataFrame (deterministic, no learning)
Estimator    — has fit(df): Transformer (learns from data, returns a fitted model)
Pipeline     — Estimator: sequence of stages, fit produces PipelineModel
PipelineModel — Transformer: fitted pipeline, ready to transform new data
```

---

## Core Concepts

```scala
import org.apache.spark.ml.{Pipeline, PipelineModel, PipelineStage}
import org.apache.spark.ml.feature._
import org.apache.spark.ml.classification._
import org.apache.spark.ml.regression._
import org.apache.spark.ml.clustering._
import org.apache.spark.ml.evaluation._
import org.apache.spark.ml.tuning._

// Define stages
val stages: Array[PipelineStage] = Array(
  new StringIndexer().setInputCol("category").setOutputCol("category_idx"),
  new OneHotEncoder().setInputCol("category_idx").setOutputCol("category_vec"),
  new VectorAssembler()
    .setInputCols(Array("price", "quantity", "category_vec"))
    .setOutputCol("features"),
  new LogisticRegression()
    .setMaxIter(100)
    .setRegParam(0.01)
    .setLabelCol("label")
    .setFeaturesCol("features")
)

val pipeline      = new Pipeline().setStages(stages)
val pipelineModel = pipeline.fit(trainData)   // learns StringIndexer mappings + LR weights
val predictions   = pipelineModel.transform(testData)

// Persist and reload
pipelineModel.save("s3://bucket/models/order-classifier/v1/")
val loaded = PipelineModel.load("s3://bucket/models/order-classifier/v1/")
```

---

## Feature Transformers (complete list)

### Text Processing

```scala
// Tokenizer — split string into words by whitespace
new Tokenizer()
  .setInputCol("text")
  .setOutputCol("words")

// RegexTokenizer — custom delimiter
new RegexTokenizer()
  .setInputCol("text")
  .setOutputCol("tokens")
  .setPattern("\\W+")   // split on non-word characters
  .setMinTokenLength(3) // ignore tokens shorter than 3 chars

// StopWordsRemover — remove common words
new StopWordsRemover()
  .setInputCol("tokens")
  .setOutputCol("filtered")
  // default English stop words; override with .setStopWords(Array(...))

// HashingTF — term frequency via hashing trick
new HashingTF()
  .setInputCol("filtered")
  .setOutputCol("tf_features")
  .setNumFeatures(1 << 18)  // hash space size (default 2^18)

// IDF — inverse document frequency (weight rare terms higher)
new IDF()
  .setInputCol("tf_features")
  .setOutputCol("tfidf_features")
  .setMinDocFreq(2)  // ignore terms in fewer than N documents

// CountVectorizer — explicit vocabulary (unlike HashingTF, provides vocabulary access)
new CountVectorizer()
  .setInputCol("filtered")
  .setOutputCol("cv_features")
  .setVocabSize(10000)
  .setMinDF(2.0)  // min document frequency

// Word2Vec — dense word embeddings
new Word2Vec()
  .setInputCol("filtered")
  .setOutputCol("word_vectors")
  .setVectorSize(100)
  .setWindowSize(5)
  .setMinCount(1)
```

### Categorical Encoding

```scala
// StringIndexer — encode string column as 0-based index (most frequent = 0)
new StringIndexer()
  .setInputCol("category")
  .setOutputCol("category_idx")
  .setHandleInvalid("keep")  // "error" (default), "skip", or "keep" (adds extra index)

// OneHotEncoder — index → sparse binary vector
new OneHotEncoder()
  .setInputCols(Array("category_idx", "day_idx"))
  .setOutputCols(Array("category_vec", "day_vec"))
  .setDropLast(true)  // drop last category to avoid multicollinearity (default true)

// IndexToString — reverse StringIndexer (for predictions back to labels)
new IndexToString()
  .setInputCol("prediction")
  .setOutputCol("predictedLabel")
  .setLabels(stringIndexerModel.labels)
```

### Numeric Transformations

```scala
// VectorAssembler — combine multiple columns into one feature vector
new VectorAssembler()
  .setInputCols(Array("price", "quantity", "category_vec", "day_vec"))
  .setOutputCol("features")
  .setHandleInvalid("skip")  // skip rows with NaN/"keep" keeps them

// StandardScaler — zero mean, unit variance (z-score normalization)
new StandardScaler()
  .setInputCol("features")
  .setOutputCol("scaled")
  .setWithMean(true)   // center to zero mean
  .setWithStd(true)    // scale to unit variance

// MinMaxScaler — scale to [min, max] range (default [0, 1])
new MinMaxScaler()
  .setInputCol("features")
  .setOutputCol("minmax_features")
  .setMin(0.0)
  .setMax(1.0)

// Normalizer — scale each row vector to unit norm
new Normalizer()
  .setInputCol("features")
  .setOutputCol("normalized")
  .setP(2.0)  // L2 norm (Euclidean); 1.0 = L1 (Manhattan)

// Bucketizer — numeric → discrete bin by custom split points
new Bucketizer()
  .setInputCol("age")
  .setOutputCol("age_bucket")
  .setSplits(Array(Double.NegativeInfinity, 18.0, 35.0, 60.0, Double.PositiveInfinity))
  // bin 0 = (−∞, 18), bin 1 = [18, 35), bin 2 = [35, 60), bin 3 = [60, ∞)

// QuantileDiscretizer — auto-compute bucket split points from data
new QuantileDiscretizer()
  .setInputCol("income")
  .setOutputCol("income_bucket")
  .setNumBuckets(10)         // percentile-based buckets
  .setRelativeError(0.001)   // precision of quantile approximation
```

### Dimensionality Reduction and Feature Selection

```scala
// PCA — principal component analysis
new PCA()
  .setInputCol("features")
  .setOutputCol("pca_features")
  .setK(50)  // keep top 50 principal components

// VectorSlicer — select a subset of features by index or name
new VectorSlicer()
  .setInputCol("features")
  .setOutputCol("selected")
  .setIndices(Array(0, 2, 5))      // by position
  // .setNames(Array("price", "qty"))  // by name (requires metadata)

// ChiSqSelector — statistical feature selection for classification
new ChiSqSelector()
  .setInputCol("features")
  .setLabelCol("label")
  .setOutputCol("selected")
  .setNumTopFeatures(50)  // keep top 50 by chi-squared score
```

---

## Estimators (supervised and unsupervised)

### Classification

```scala
// Logistic Regression
new LogisticRegression()
  .setMaxIter(100)
  .setRegParam(0.01)          // L1+L2 regularization strength
  .setElasticNetParam(0.0)    // 0 = L2 (Ridge), 1 = L1 (Lasso), between = ElasticNet
  .setThreshold(0.5)          // decision threshold for binary classification
  .setFamily("binomial")      // "binomial" | "multinomial" | "auto"

// Decision Tree
new DecisionTreeClassifier()
  .setMaxDepth(5)
  .setImpurity("gini")        // "gini" | "entropy"
  .setMinInstancesPerNode(1)
  .setMaxBins(32)             // number of bins for continuous features

// Random Forest
new RandomForestClassifier()
  .setNumTrees(100)
  .setMaxDepth(5)
  .setFeatureSubsetStrategy("auto")  // "auto" | "all" | "sqrt" | "log2" | fraction
  .setSeed(42L)

// Gradient Boosted Trees
new GBTClassifier()
  .setMaxIter(20)             // number of trees
  .setStepSize(0.1)           // learning rate
  .setMaxDepth(5)
  .setLossType("logistic")    // "logistic" for binary classification

// Naive Bayes (text classification)
new NaiveBayes()
  .setSmoothing(1.0)          // Laplace smoothing
  .setModelType("multinomial") // "multinomial" | "bernoulli" | "gaussian"
```

### Regression

```scala
// Linear Regression
new LinearRegression()
  .setMaxIter(100)
  .setRegParam(0.01)
  .setElasticNetParam(0.0)
  .setLoss("squaredError")    // "squaredError" | "huber"

// Random Forest Regressor
new RandomForestRegressor()
  .setNumTrees(50)
  .setMaxDepth(5)

// Gradient Boosted Trees Regressor
new GBTRegressor()
  .setMaxIter(50)
  .setStepSize(0.1)
  .setLossType("squared")     // "squared" | "absolute"
```

### Clustering

```scala
// KMeans
new KMeans()
  .setK(10)                   // number of clusters
  .setMaxIter(20)
  .setInitMode("k-means||")   // "k-means||" (default) | "random"
  .setSeed(42L)

// result: model.clusterCenters — Array[Vector]
// prediction column: "prediction" (cluster index 0..K-1)

// Bisecting KMeans (hierarchical, faster for large K)
new BisectingKMeans()
  .setK(10)
  .setMaxIter(20)

// LDA (topic modelling — unsupervised)
new LDA()
  .setK(10)                   // number of topics
  .setMaxIter(20)
  .setOptimizer("em")         // "em" | "online"
  .setTopicDistributionCol("topicDistribution")
```

---

## Model Evaluation

```scala
// Binary classification
val binEval = new BinaryClassificationEvaluator()
  .setLabelCol("label")
  .setRawPredictionCol("rawPrediction")
  .setMetricName("areaUnderROC")   // "areaUnderROC" | "areaUnderPR"

val auc = binEval.evaluate(predictions)

// Multi-class classification
val mcEval = new MulticlassClassificationEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")
  .setMetricName("f1")  // "f1" | "accuracy" | "weightedPrecision" | "weightedRecall"

// Regression
val regEval = new RegressionEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")
  .setMetricName("rmse")  // "rmse" | "mse" | "r2" | "mae" | "var"

// Clustering
val clusterEval = new ClusteringEvaluator()
  .setMetricName("silhouette")  // "silhouette" — higher is better
```

---

## Cross-Validation and Hyperparameter Tuning

```scala
// Build the parameter grid
val paramGrid = new ParamGridBuilder()
  .addGrid(lr.regParam,        Array(0.001, 0.01, 0.1, 1.0))
  .addGrid(lr.elasticNetParam, Array(0.0, 0.5, 1.0))
  .addGrid(lr.maxIter,         Array(50, 100))
  .build()
// Total: 4 × 3 × 2 = 24 combinations

// K-fold cross-validation (thorough, expensive)
val cv = new CrossValidator()
  .setEstimator(pipeline)
  .setEvaluator(new BinaryClassificationEvaluator())
  .setEstimatorParamMaps(paramGrid)
  .setNumFolds(5)
  .setParallelism(4)   // fit up to 4 folds in parallel

val cvModel = cv.fit(trainData)
cvModel.avgMetrics   // Array[Double] — average metric per param combo
cvModel.bestModel    // PipelineModel with best params

// Train-validation split (faster — single 80/20 split instead of K folds)
val tvs = new TrainValidationSplit()
  .setEstimator(pipeline)
  .setEvaluator(new BinaryClassificationEvaluator())
  .setEstimatorParamMaps(paramGrid)
  .setTrainRatio(0.8)
  .setParallelism(4)

val tvsModel = tvs.fit(trainData)
```

---

## Save, Load, and Versioning

```scala
// Save
pipelineModel.save("s3://bucket/models/my-model/v1/")

// Load (returns PipelineModel)
val loaded = PipelineModel.load("s3://bucket/models/my-model/v1/")

// Overwrite version
pipelineModel.write.overwrite().save("s3://bucket/models/my-model/v2/")

// Extract a specific stage from a loaded model (e.g., to inspect feature importances)
val rfModel = loaded.stages(3).asInstanceOf[RandomForestClassificationModel]
rfModel.featureImportances  // SparseVector of feature importance scores
```

---

## TDD for MLlib Pipelines

```scala
class PipelineSpec extends AnyFunSpec with Matchers with BeforeAndAfterAll {
  lazy val spark = SparkSession.builder().master("local[2]").appName("ml-test").getOrCreate()
  import spark.implicits._

  override def afterAll(): Unit = spark.stop()

  // Level 1: individual transformer produces expected output
  it("StringIndexer assigns 0 to most frequent category") {
    val data = Seq(("A",), ("A",), ("B",)).toDF("category")
    val indexer = new StringIndexer().setInputCol("category").setOutputCol("idx")
    val result  = indexer.fit(data).transform(data)
    result.filter($"category" === "A").select("idx").as[Double].head() shouldBe 0.0
  }

  // Level 2: pipeline produces required output columns
  it("pipeline outputs prediction and probability columns") {
    val preds = trainedModel.transform(testData)
    preds.columns should contain allOf ("prediction", "probability", "rawPrediction")
  }

  // Level 3: model beats minimum quality threshold
  it("AUC exceeds 0.7 on held-out test set") {
    val evaluator = new BinaryClassificationEvaluator().setMetricName("areaUnderROC")
    evaluator.evaluate(trainedModel.transform(testData)) should be > 0.7
  }

  // Level 4: serialization round-trip
  it("loaded model produces identical predictions") {
    val tmpPath = Files.createTempDirectory("ml-test").toString
    trainedModel.save(tmpPath)
    val loaded = PipelineModel.load(tmpPath)
    val original = trainedModel.transform(testData).select("prediction").collect()
    val reloaded = loaded.transform(testData).select("prediction").collect()
    original shouldBe reloaded
  }
}
```

---

## Common Pitfalls

```scala
// Pitfall 1: fitting on test data (data leakage)
// BAD:
val model = pipeline.fit(allData)
// GOOD:
val Array(trainData, testData) = allData.randomSplit(Array(0.8, 0.2), seed = 42L)
val model = pipeline.fit(trainData)  // fit only on training data

// Pitfall 2: StringIndexer fails on unseen categories
// Set handleInvalid = "keep" or "skip" for production data that may have new categories

// Pitfall 3: VectorAssembler drops rows with NaN silently
// Set setHandleInvalid("skip") explicitly, or impute missing values first
new Imputer()
  .setInputCols(Array("price", "quantity"))
  .setOutputCols(Array("price_imp", "qty_imp"))
  .setStrategy("mean")  // "mean" | "median" | "mode"

// Pitfall 4: forgetting to set seed for reproducibility
new RandomForestClassifier().setSeed(42L)  // always set seed for reproducible results
new KMeans().setSeed(42L)
```
