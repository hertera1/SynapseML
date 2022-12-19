---
title: LightGBM
hide_title: true
sidebar_label: About
---

# LightGBM on Apache Spark

### LightGBM

[LightGBM](https://github.com/Microsoft/LightGBM) is an open-source,
distributed, high-performance gradient boosting (GBDT, GBRT, GBM, or
MART) framework. This framework specializes in creating high-quality and
GPU enabled decision tree algorithms for ranking, classification, and
many other machine learning tasks. LightGBM is part of Microsoft's
[DMTK](http://github.com/microsoft/dmtk) project.

### Advantages of LightGBM

-   **Composability**: LightGBM models can be incorporated into existing
    SparkML Pipelines, and used for batch, streaming, and serving
    workloads.
-   **Performance**: LightGBM on Spark is 10-30% faster than SparkML on
    the Higgs dataset, and achieves a 15% increase in AUC.  [Parallel
    experiments](https://github.com/Microsoft/LightGBM/blob/master/docs/Experiments.rst#parallel-experiment)
    have verified that LightGBM can achieve a linear speed-up by using
    multiple machines for training in specific settings.
-   **Functionality**: LightGBM offers a wide array of [tunable
    parameters](https://github.com/Microsoft/LightGBM/blob/master/docs/Parameters.rst),
    that one can use to customize their decision tree system. LightGBM on
    Spark also supports new types of problems such as quantile regression.
-   **Cross platform** LightGBM on Spark is available on Spark, PySpark, and SparklyR

### Usage

In PySpark, you can run the `LightGBMClassifier` via:

```python
from synapse.ml.lightgbm import LightGBMClassifier
model = LightGBMClassifier(learningRate=0.3,
                           numIterations=100,
                           numLeaves=31).fit(train)
```

Similarly, you can run the `LightGBMRegressor` by setting the
`application` and `alpha` parameters:

```python
from synapse.ml.lightgbm import LightGBMRegressor
model = LightGBMRegressor(application='quantile',
                          alpha=0.3,
                          learningRate=0.3,
                          numIterations=100,
                          numLeaves=31).fit(train)
```

For an end to end application, check out the LightGBM [notebook
example](../LightGBM%20-%20Overview).

### Arguments/Parameters

SynapseML exposes getters/setters for many common LightGBM parameters.
In python, you can use the properties as shown above, or in Scala use the
fluent setters.
```scala
import com.microsoft.azure.synapse.ml.lightgbm.LightGBMClassifier
val classifier = new LightGBMClassifier()
                       .setLearningRate(0.2)
                       .setNumLeaves(50)
```

LightGBM has far more parameters than SynapseML exposes. For cases where you
need to set some parameters that SyanpseML does not expose a setter for, use
passThroughArgs. This is just a free string that you can use to add extra parameters
to the command SynapseML sends to configure LightGBM.

In python:
```python
from synapse.ml.lightgbm import LightGBMClassifier
model = LightGBMClassifier(passThroughArgs="force_row_wise=true min_sum_hessian_in_leaf=2e-3",
                           numIterations=100,
                           numLeaves=31).fit(train)
```

In Scala:
```scala
import com.microsoft.azure.synapse.ml.lightgbm.LightGBMClassifier
val classifier = new LightGBMClassifier()
                      .setPassThroughArgs("force_row_wise=true min_sum_hessian_in_leaf=2e-3")
                      .setLearningRate(0.2)
                      .setNumLeaves(50)
```

For formatting options and specific argument documentation, see
[LightGBM docs](https://lightgbm.readthedocs.io/en/v3.3.2/Parameters.html). Some
parameters SynapseML will set specifically for the Spark distributed environment and
should not be changed. Some parameters are for cli mode only, and will not work within
Spark.

Note that you can mix passThroughArgs and explicit args, as shown above. SynapseML will
merge them to create one argument string to send to LightGBM. If you set a parameter in
both places, the passThroughArgs will take precedence.

### Architecture

LightGBM on Spark uses the Simple Wrapper and Interface Generator (SWIG)
to add Java support for LightGBM. These Java Binding use the Java Native
Interface call into the [distributed C++
API](https://github.com/Microsoft/LightGBM/blob/master/include/LightGBM/c_api.h).

We initialize LightGBM by calling
[`LGBM_NetworkInit`](https://github.com/Microsoft/LightGBM/blob/master/include/LightGBM/c_api.h)
with the Spark executors within a MapPartitions call. We then pass each
workers partitions into LightGBM to create the in-memory distributed
dataset for LightGBM.  We can then train LightGBM to produce a model
that can then be used for inference.

The `LightGBMClassifier` and `LightGBMRegressor` use the SparkML API,
inherit from the same base classes, integrate with SparkML pipelines,
and can be tuned with [SparkML's cross
validators](https://spark.apache.org/docs/latest/ml-tuning.html).

Models built can be saved as SparkML pipeline with native LightGBM model
using `saveNativeModel()`. Additionally, they are fully compatible with [PMML](https://en.wikipedia.org/wiki/Predictive_Model_Markup_Language) and
can be converted to PMML format through the
[JPMML-SparkML-LightGBM](https://github.com/alipay/jpmml-sparkml-lightgbm) plugin.

### Barrier Execution Mode

By default LightGBM uses regular spark paradigm for launching tasks and communicates with the driver to coordinate task execution.
The driver thread aggregates all task host:port information and then communicates the full list back to the workers in order for NetworkInit to be called.
This requires the driver to know how many tasks there are, and if the expected number of tasks is different from actual this will cause the initialization to deadlock.
There is a new UseBarrierExecutionMode flag, which when activated uses the barrier() stage to block all tasks.
The barrier execution mode simplifies the logic to aggregate host:port information across all tasks.
To use it in scala, you can call setUseBarrierExecutionMode(true), for example:

    val lgbm = new LightGBMClassifier()
        .setLabelCol(labelColumn)
        .setObjective(binaryObjective)
        .setUseBarrierExecutionMode(true)
    ...
    <train classifier>