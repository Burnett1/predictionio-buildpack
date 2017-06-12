# Deploy [PredictionIO](http://predictionio.incubator.apache.org) to Heroku

👓 Requires intermediate technical skills working with PredictionIO, the Scala programming language, and Heroku.

🍎 For an simpler demo of PredictionIO, try the [example Predictive Classification app](https://github.com/heroku/predictionio-engine-classification).

🗺 See the [buildpack README](README.md) for an overview of the tools used in these docs.

🛠 Follow the [local development](DEV.md) workflow to setup an engine on your computer.

## Docs 📚

✏️ Throughout this document, code terms that start with `$` represent a value (shell variable) that should be replaced with a customized value, e.g `$EVENTSERVER_NAME`, `$ENGINE_NAME`, `$POSTGRES_ADDON_ID`…

Please, follow the steps in the order documented.

* [Eventserver](#eventserver)
  1. [Create the eventserver](#create-the-eventserver)
  1. [Deploy the eventserver](#deploy-the-eventserver)
* [Engine](#engine)
  1. [Create an engine](#create-an-engine)
     * [Optional Persistent Filesystem](#optional-persistent-filesystem)
  1. [Create a Heroku app for the engine](#create-a-heroku-app-for-the-engine)
  1. [Configure the Heroku app to use the eventserver](#configure-the-heroku-app-to-use-the-eventserver)
  1. [Update source configs](#update-source-configs)
     1. [`template.json`](#update-template-json)
     1. [`build.sbt`](#update-build-sbt)
     1. [`engine.json`](#update-engine-json)
  1. [Import data](#import-data)
     * [Built-in Data Hooks](#built-in-data-hooks)
  1. [Deploy to Heroku](#deploy-to-heroku)
     * [Scale-up](#scale-up)
     * [Retry release](#retry-release)
* [Training](#training)
  * [Automatic training](#automatic-training)
  * [Manual training](#manual-training)
* [Evaluation](#evaluation)
  1. [Changes required for evaluation](#changes-required-for-evaluation)
  1. [Perform evaluation](#perform-evaluation)
  1. [Re-deploy best parameters](#re-deploy-best-parameters)
* [Configuration](#configuration)
  * [Environment variables](#environment-variables)
  * [`pio-env.sh` and other config files](#pio-env-sh-and-other-config-files)
* [Local development](#local-development)
  * [`pio-shell`](#pio-shell)
* [Testing](#testing)


## Eventserver

### Create the eventserver

⚠️ **Each engine should have its own eventserver.** It's *possible* to share a event storage between engines only if they share the same storage backends and configuration. Otherwise, various storage-related errors will emerge and break the engine. *This is a change from the previous advice given here.*

[![Deploy Eventserver](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy) with free Heroku Postgres database (limit 10K rows).

If you need a larger (greater that 10K rows), then provision the eventserver manually on a paid database tier:

```bash
git clone https://github.com/heroku/predictionio-buildpack.git pio-eventserver
cd pio-eventserver

heroku create $EVENTSERVER_NAME
heroku buildpacks:set https://github.com/heroku/predictionio-buildpack
heroku addons:create heroku-postgresql:hobby-basic
```

Note the Postgres add-on identifier, e.g. `postgresql-aerodynamic-00000`; use it below in place of `$POSTGRES_ADDON_ID`

We delay deployment until the database is ready:

```bash
heroku pg:wait && git push heroku master
```


## Engine

Select an engine from the [gallery](https://predictionio.incubator.apache.org/gallery/template-gallery/). Download a `.tar.gz` from Github and open/expand it on your local computer.

🏷 This buildpack should be used with engine templates for **PredictionIO 0.11**.

### Create an engine

`cd` into the engine's directory, and ensure it is a git repo:

```bash
git init
```

### Create a Heroku app for the engine

```bash
heroku create $ENGINE_NAME
heroku buildpacks:set https://github.com/heroku/predictionio-buildpack
```

### Optional Persistent Filesystem

👓 Heroku dynos have an [ephemeral filesystem](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem).

For engines that require filesystem persistence, this buildpack supports [HDFS](https://en.wikipedia.org/wiki/Apache_Hadoop#HDFS) on [Amazon S3](https://aws.amazon.com/s3/).

To enable, either:

* use the [S3 Add-on](https://devcenter.heroku.com/articles/bucketeer) ($5/month minimum cost)

  ```bash
  heroku addons:create bucketeer --as PIO_S3
  ```
* bring your own [s3 bucket](https://aws.amazon.com/s3/) by manually setting the [config vars](#environment-variables)
  * `PIO_S3_BUCKET_NAME`
  * `PIO_S3_AWS_ACCESS_KEY_ID`
  * `PIO_S3_AWS_SECRET_ACCESS_KEY`

⚠️ Note that with HDFS on Heroku, all filesystem path references must be absolute from `/` root, not relative or nested in User ID directory.

### Configure the Heroku app to use the eventserver

⚠️ **Not required for engines that exclusively use a custom data source.**

Replace the Postgres ID & eventserver config values with those from above:

```bash
heroku addons:attach $POSTGRES_ADDON_ID

heroku config:set PIO_EVENTSERVER_APP_NAME=$PIO_APP_NAME
# must match `appName` in engine.json & eventserver
```

* See [environment variables](#environment-variables) for config details.

### Update source configs

#### `template.json`

The version of PredictionIO used for deployment is based in the value in this file:

```json
  "pio": {
    "version" : {
      "min": "0.11.0-incubating"
    }
  }
```

#### `build.sbt`

The Scala built tool config must be updated with Scala, PredictionIO, & Spark versions:

```scala
scalaVersion := "2.11.8"

organization := "org.apache.predictionio"

libraryDependencies ++= Seq(
  "org.apache.predictionio" %% "apache-predictionio-core" % "0.11.0-incubating" % "provided",
  "org.apache.spark"        %% "spark-core"               % "2.1.0" % "provided",
  "org.apache.spark"        %% "spark-mllib"              % "2.1.0" % "provided")
```

#### `engine.json`

Update so the `appName` parameter matches the [value set for `PIO_EVENTSERVER_APP_NAME`](#configure-the-heroku-app-to-use-the-eventserver).

```json
  "datasource": {
    "params" : {
      "appName": "$PIO_EVENTSERVER_APP_NAME"
    }
  }
```

⭐️  **A better alternative** is to delete the `"appName"` param from `engine.json`, and then use the environment variable value `sys.env("PIO_EVENTSERVER_APP_NAME")` in the engine source code.

### Import data

🚨 **Mandatory: Data is required.** The first time an engine is deployed, it requires data for training.

⚠️  If `data/initial-events.json` already exists in the engine, then skip to [Deploy to Heroku](#deploy-to-heroku). This data will automatically be imported into the eventserver before training.

Many community-contributed engine templates provide a Python `data/import_events.py` script which may be run manually from a local machine to load data via the Eventserver's REST API. While popular for getting an example running, this method is not optimum for Heroku deployment workflow, because it requires a Python installation (nothing else in PredictionIO uses Python), it limits import performance through the Eventserver web process (extra complexity of running & scaling that process), and it is not transactional (individual REST failures will not fail the process).

#### Built-in Data Hooks

With this buildpack, initial data import and ongoing synchronization may be automated using script hooks to generate JSON data that is automatically imported before training using `pio import`, an efficient method using concurrent database connections.

🔍 See the [Data Flow docs](DATA.md) for how to leverage the built-in import & sync workflow.

### Deploy to Heroku

```bash
git add .
git commit -m "Initial PIO engine"
git push heroku master

# Follow the logs to see training 
# and then start-up of the engine.
#
heroku logs -t --app $ENGINE_NAME
```

⚠️ **Initial deploy will probably fail due to memory constraints.** To fix, [scale up](#scale-up) and [retry the release](#retry-release).

#### Scale up

Once deployed, scale up the processes and config Spark to avoid memory issues. These are paid, [professional dyno types](https://devcenter.heroku.com/articles/dyno-types#available-dyno-types):

```bash
heroku ps:scale \
  web=1:Performance-M \
  release=0:Performance-L \
  train=0:Performance-L
```

#### Retry release

When the release (`pio train`) fails due to memory constraints or other transient error, you may use the Heroku CLI [releases:retry plugin](https://github.com/heroku/heroku-releases-retry) to rerun the release without pushing a new deployment.


## Training

### Automatic training

`pio train` will automatically run during [release-phase of the Heroku app](https://devcenter.heroku.com/articles/release-phase).

### Manual training

```bash
heroku run train

# You may need to revive the app from "crashed" state.
heroku restart
```

## Evaluation

PredictionIO provides an [Evaluation mode for engines](https://predictionio.incubator.apache.org/evaluation/), which uses cross-validation to help select optimum engine parameters.

⚠️ Only engines that contain `src/main/scala/Evaluation.scala` support Evaluation mode.

### Changes required for evaluation

To run evaluation on Heroku, ensure `src/main/scala/Evaluation.scala` references the engine's name through the environment. Check the source file to verify that `appName` is set to `sys.env("PIO_EVENTSERVER_APP_NAME")`. For example:

```scala
DataSourceParams(appName = sys.env("PIO_EVENTSERVER_APP_NAME"), evalK = Some(5))
```

♻️ If that change was made, then commit, deploy, & re-train before proceeding.

### Perform evaluation

Next, start a console & change to the engine's directory. This uses a paid, [professional dyno type](https://devcenter.heroku.com/articles/dyno-types#available-dyno-types):

```bash
heroku run bash --size Performance-L
$ cd pio-engine/
```

Then, start the process, specifying the evaluation & engine params classes from the `Evaluation.scala` source file. For example:

```bash
$ pio eval \
    org.template.classification.AccuracyEvaluation \
    org.template.classification.EngineParamsList  \
    -- $PIO_SPARK_OPTS
```

✏️ Memory parameters are set to fit the [dyno `--size`](https://devcenter.heroku.com/articles/dyno-types#available-dyno-types) set in the `heroku run` command.

### Re-deploy best parameters

Once `pio eval` completes, still in the Heroku console, copy the contents of `best.json`:

```bash
$ cat best.json
```

♻️ Paste into your local `engine.json`, commit, & deploy.


## Configuration

### Environment variables

Engine deployments honor the following config vars:

* `DATABASE_URL` & `PIO_POSTGRES_OPTIONAL_SSL`
  * automatically set by [Heroku PostgreSQL](https://elements.heroku.com/addons/heroku-postgresql)
  * defaults to `postgres://pio:pio@locahost:5432/pio`
  * when testing locally, set `PIO_POSTGRES_OPTIONAL_SSL=true` to avoid **The server does not support SSL** errors
* `PIO_VERBOSE`
  * set `PIO_VERBOSE=true` for detailed build logs
* `PIO_MAVEN_REPO`
  * add a Maven repository URL to search when installing deps from engine's `build.sbt`
  * useful for testing pre-release packages
* `PREDICTIONIO_DIST_URL`
  * defaults to a PredictionIO distribution version based on `pio.version.min` in **template.json**
  * use a custom distribution by setting its fetch URL:

    ```bash
    heroku config:set PREDICTIONIO_DIST_URL=https://marsikai.s3.amazonaws.com/PredictionIO-0.10.0-incubating.tar.gz
    ```
* `PIO_OPTS`
  * options passed as `pio $opts`
  * see: [`pio` command reference](https://predictionio.incubator.apache.org/cli/)
  * example:

    ```bash
    heroku config:set PIO_OPTS='--variant best.json'
    ```
* `PIO_SPARK_OPTS` & `PIO_TRAIN_SPARK_OPTS`
  * **deploy** & **training** options passed through to `spark-submit $opts`
  * see: [`spark-submit` reference](http://spark.apache.org/docs/1.6.1/submitting-applications.html)
  * example, overriding the automatic (fit-to-dyno) Spark memory settings:

    ```bash
    heroku config:set \
      PIO_SPARK_OPTS='--executor-memory 1536m --driver-memory 1g' \
      PIO_TRAIN_SPARK_OPTS='--executor-memory 10g --driver-memory 4g'
    ```
  * example, using an existing Spark cluster (deploying a cluster is outside the scope of the buildpack):

    ```bash
    heroku config:set \
      PIO_TRAIN_SPARK_OPTS='--master spark://my-master.example.com:7077' \
      PIO_SPARK_OPTS='--master spark://my-master.example.com:7077'
    ```
  * note this additional constraint of Spark pass-through args,  
    `spark.driver.extraJavaOptions` is silently ignored:

    ```bash
    # Options are silently dropped when set through `--conf`.
    # Bad example; don't use this:
    PIO_SPARK_OPTS="--conf 'spark.driver.extraJavaOptions=-Dcom.amazonaws.services.s3.enableV4'"

    # Instead, pass them using `--driver-java-options`.
    # Good example; do this:
    PIO_SPARK_OPTS="--driver-java-options '-Dcom.amazonaws.services.s3.enableV4'"
    ```
* `PIO_ENABLE_FEEDBACK`
  * set `PIO_ENABLE_FEEDBACK=true` to enable feedback loop; auto-generation of historical prediction events for analysis of engine performance
  * requires the `PIO_EVENTSERVER_*` vars to be configured
* `PIO_EVENTSERVER_HOSTNAME` & `PIO_EVENTSERVER_PORT`
  * `$EVENTSERVER_NAME.herokuapp.com` & `443` (default) for Heroku apps' HTTPS interface
* `PIO_EVENTSERVER_APP_NAME` & `PIO_EVENTSERVER_ACCESS_KEY`
  * may be generated by running `pio app new $PIO_APP_NAME` on the eventserver
* `PIO_PURGE_EVENTS_ON_SYNC`
  * set `PIO_PURGE_EVENTS_ON_SYNC=true` to delete all existing events before each data import
* `PIO_TRAIN_ON_RELEASE`
  * set `false` to disable automatic training
  * subsequent deploys may crash a deployed engine until it's retrained
  * use [manual training](#manual-training)
* `PIO_S3_BUCKET_NAME`, `PIO_S3_AWS_ACCESS_KEY_ID`, & `PIO_S3_AWS_SECRET_ACCESS_KEY`
  * configures a bucket to enable filesystem access
* `AWS_REGION`
  * when connecting to S3 in region other than US, the region name must be specified to enable signature v4.
* `PIO_ELASTICSEARCH_URL`
  * when set, activates [Elasticsearch](https://www.elastic.co/products/elasticsearch) as the [metadata store](http://predictionio.incubator.apache.org/system/anotherdatastore/)
  * Elasticsearch version 5.x is supported
  * use an [add-on](https://elements.heroku.com/search/addons?q=elasticsearch):

    ```bash
    heroku addons:create bonsai --version 5.1 --as PIO_ELASTICSEARCH
    ```

### `pio-env.sh` and other config files

The buildpack comes with [`config/`](config/) ERB templates that are rendered using the current [environment variables](#environment-variables) when the app is launched. Any one of these may be customized by creating a `config/` directory in your engine and copying over the template from this buildpack. Use caution when making modifications, as these configs are preset to work on Heroku.

## Local development

▶️ setup an engine for the [local development](DEV.md) workflow

### `pio-shell`

Use the interactive Scala REPL to work with an engine locally.

```bash
pio-shell \
  --with-spark \
  --jars PredictionIO-dist/lib/pio-assembly-0.11.0-SNAPSHOT.jar,PredictionIO-dist/lib/postgresql_jdbc.jar,PredictionIO-dist/lib/spark/pio-data-elasticsearch-assembly-0.11.0-SNAPSHOT.jar,PredictionIO-dist/lib/spark/pio-data-jdbc-assembly-0.11.0-SNAPSHOT.jar
```

(This following command includes a `--jars` fix for PIO 0.11.0-incubating. If you're not using [local development](DEV.md) workflow, then those paths will be different for your own setup.)

Then, load the necessary classes to load some event data:

```scala
scala> import org.apache.predictionio.data.store.PEventStore
scala> PEventStore.aggregateProperties(appName="my-app", entityType="user")(sc).collect { case(i,p) => i }.take(5).foreach(println)
```

## Testing

### Buildpack [![Build Status](https://travis-ci.org/heroku/predictionio-buildpack.svg?branch=master)](https://travis-ci.org/heroku/predictionio-buildpack)

[Tests](test/) covering this buildpack's build and release functionality are implemented with [heroku-buildpack-testrunner](https://github.com/heroku/heroku-buildpack-testrunner). Engine test cases are staged in the [`test/fixtures/`](test/fixtures/).

Setup [testrunner with Docker](https://github.com/heroku/heroku-buildpack-testrunner#docker-usage), then run tests with:

```bash
docker-compose -p pio -f test/docker-compose.yml run testrunner
```

### Individual Apps

Engines deployed as Heroku apps may automatically run their `sbt test` suite using [Heroku CI (beta)](https://devcenter.heroku.com/articles/heroku-ci):

>Heroku CI automatically runs tests for every subsequent push to your GitHub repository. Any push to any branch triggers a test run, including a push to master. This means that all GitHub pull requests are automatically tested, along with any merges to master.
>
> Test runs are executed inside an ephemeral Heroku app that is provisioned for the test run. The app is destroyed when the run completes.
