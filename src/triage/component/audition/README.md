# Audition

Choosing the best classifier models

[![Build Status](https://travis-ci.org/dssg/audition.svg?branch=master)](https://travis-ci.org/dssg/audition)
[![codecov](https://codecov.io/gh/dssg/audition/branch/master/graph/badge.svg)](https://codecov.io/gh/dssg/audition)
[![codeclimate](https://codeclimate.com/github/dssg/audition.png)](https://codeclimate.com/github/dssg/audition)

## Overview
Audition is a tool for picking the best trained classifiers from a predictive analytics experiment. Often, production-scale experiments will come up with thousands of trained models, and sifting through all of those results can be time-consuming even after calculating the usual basic metrics like precision and recall. Which metrics matter most? Should you prioritize the best metric value over time or treat recent data as most important? Is low metric variance important? The answers to questions like these may not be obvious up front. Audition introduces a structured, semi-automated way of filtering models based on what you consider important, with an interface that is easy to interact with from a Jupyter notebook (with plots), but is driven by configuration that can easily be scripted.

### Use Case
We’ve just built hundreds of models over time - each model group trains on a given train set and scores the test set and calculates several metrics.

### What does it do?
Input: 
* List of models built over different train periods and tested on different test periods
* Model selection rules
* Metric(s) of interest

Process:
1. Gets rid of really bad model groups wrt the metric of interest. A model group is discarded if:
* It’s never close to the “best” model (define close to best) or
* If it’s metric is below a certain number (define min threshold)  at least once
We iterate over various values of these parameters to end up with a reasonable number of model groups to pass ot the next step

2. Apply existing (or new, custom) selection rules to the model groups passed from step 1. Current supported rules are “best_most_recent”, “best_average” for one or two metrics, best_recent_average, “lowest_variance”, high_avg_low_variance, “most_frequent_best_distance”
a. For each rule
For each time period
It applies the rule to select a model for that time period (based on performance of that model so far)
It calculates the evaluation metric for that selected model on the next time period
calculates regret (how much worse is the selected model compared to the best model in the next time period)
Absolute value
rank/percentile [todo]
Now we have a regret for each rule for each time period. We now have to decide on which rule to use
Do all/most rules pick the same/similar model groups?
If so, then audition should output those model groups

Output:
* Top k model groups for each rule and average regret for each rule

## Preparing Experiment Results

Note: If you are familiar with the DSaPP 'results schema', you can skip this section and head to the [Using the Auditioner](#using) section.

Audition expects to be able to read experiment metadata from a relational database. This includes information about models, what we call 'model groups', and evaluations. The full experiment schema used by DSaPP post-modeling tools such as Audition is defined in the [results-schema](github.com/dssg/results-schema) repository, and is automatically populated after a [triage.Experiment](github.com/dssg/triage). However, even without using those tools, you can populate these tables for other experiments. Here's an overview of Audition's direct dependencies:

* `results.model_groups` - Everything unique about a classifier model, except for its train date. We define this as:
	* `model_group_id` - An autogenerated integer surrogate key for the model group, used as a foreign key in other tables
	* `model_type` - The name of the class, e.g 'sklearn.ensemble.RandomForestClassifier'
	* `model_parameters` - Often referred to as hyperparameters. This is a dictionary (stored in the database as JSON) that describes how the class is configured (e.g. {'criterion': 'gini', 'min_samples_split': 2})
	* `feature_list` - A list of feature names (stored in the database as an array).
	* `model_config` - A catch-all for anything else you want to uniquely define as a model group, for instance different pieces of time config. Use of this column is not strictly necessary.
* `results.models` - An instantiation of a model group at a specific training time. In production, our version of this table has quite a few column, but the only columns used by Audition are below. If you reproduce this table, these three should be sufficient:
	* `model_id` - An autogenerated integer surrogate key for the model, used as a foreign key in the 'evaluations' table.
	* `model_group_id` - A foreign key to the model_groups table.
	* `train_end_time` - A timestamp that signifies when this model was trained. 
* `results.evaluations` - This is where metrics (such as precision, recall) and their values for specific models get stored.
	* `model_id` - A foreign key to the models table
	* `evaluation_start_time` - The start of the time period from which testing data was taken from.
	* `evaluation_end_time` - The end of the time period from which testing data was taken from.
	* `metric` - The name of a metric, e.g. 'precision@' for thresholded precision. Audition needs some knowledge about what direction indicates 'better' results for this metric, so it wishes that the metric is one of the options in [catwalk.ModelEvaluator.available_metrics](github.com/dssg/catwalk/blob/master/catwalk/evaluation.py#L43). If you use a metric that is not available here, it will assume that greater is better.
	* `parameter` - A string that indicates any parameters for the metric. For instance, `100_abs` indicates top-100 entities, while `5_pct` indicates top 5 percentile of entities. These are commonly used for metrics like precision and recall to prioritize the evaluation of models to how they affect the actions likely to be taken as a result of the model.
	* `value` - A float that represents the value of the metric and parameters applied to the model and evaluation time.

## <a name="using"></a>Using the Auditioner

### Setting Up the Auditioner
The primary interface for Audition is the `Auditioner` class, and a recommended way to start using it is within a Jupyter notebook so you can see plots.

The Auditioner requires a few inputs to get started:
* db_engine (sqlalchemy.engine) A database engine with access to a results schema of a completed modeling run
* model_group_ids (list) A large list of model groups to audition. No effort should
	be needed to pick 'good' model groups, but they should all be groups that could
	be used if they are found to perform well. They should also each have evaluations
	for any train end times you wish to include in analysis
* train_end_times (list) A list of train end times that all of the given model groups
	contain evaluations for and that you want to be deemed important in the analysis
* initial_metric_filters (list) A list of metrics to filter model groups on, and how to filter them. These are all expected to be in the evaluations table already for all specified models. Each entry should be a dict with the keys:
	* metric (string) -- model evaluation metric, such as 'precision@'
	* parameter (string) -- model evaluation metric parameter,
		such as '300_abs'
	* max_from_best (float) The maximum value that the given metric can be worst the best for a given train end time. To start out without filtering, this could be the theoretical maximum range of the metric, often '1.0'.
	* threshold_value (float) The worst absolute value that the given metric should be. Depending on the metric, this could be a minimum (precision) or a maximum (false positives)


Example: (coming soon)

### Iterating on the Metric Filters
(coming soon)

### Passing a Selection Rule Grid
(coming soon)

### Exporting results
(coming soon)