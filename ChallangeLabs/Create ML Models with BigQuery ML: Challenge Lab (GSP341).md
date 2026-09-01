# Create ML Models with BigQuery ML: Challenge Lab (GSP341)

Here is the complete, single-block execution script formatted as Markdown.

### How to use this solution:

1. Open the **Google Cloud Console**.
2. Launch **Cloud Shell**.
3. Copy the entire code block below.
4. Paste it directly into the Cloud Shell terminal and press **Enter**.
5. Wait for the models to finish training (each model takes about 2 to 5 minutes to build).
6. When the script completes, click **Check my progress** for all tasks on the lab page to receive 100/100 points!

---

### Execution Script

```bash
# 1. Enable BigQuery API (usually enabled by default, but safe to run)
gcloud services enable bigquery.googleapis.com

# 2. Task 1: Create dataset and initial customer_classification_model
echo "Creating ecommerce dataset..."
bq mk --dataset ecommerce

echo "Training customer_classification_model (This takes a few minutes)..."
bq query --use_legacy_sql=false "
CREATE OR REPLACE MODEL \`ecommerce.customer_classification_model\`
OPTIONS
(
model_type='logistic_reg',
labels = ['will_buy_on_return_visit']
)
AS
SELECT
  * EXCEPT(fullVisitorId)
FROM
  (SELECT
    fullVisitorId,
    IFNULL(totals.bounces, 0) AS bounces,
    IFNULL(totals.timeOnSite, 0) AS time_on_site
  FROM
    \`data-to-insights.ecommerce.web_analytics\`
  WHERE
    totals.newVisits = 1
    AND date BETWEEN '20160801' AND '20170430')
JOIN
  (SELECT
    fullvisitorid,
    IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM
    \`data-to-insights.ecommerce.web_analytics\`
  GROUP BY fullvisitorid)
USING (fullVisitorId);
"

# 3. Task 2: Evaluate the customer_classification_model against unseen data
echo "Evaluating customer_classification_model..."
bq query --use_legacy_sql=false "
SELECT
  roc_auc,
  accuracy
FROM
  ML.EVALUATE(MODEL \`ecommerce.customer_classification_model\`, (
SELECT
  * EXCEPT(fullVisitorId)
FROM
  (SELECT
    fullVisitorId,
    IFNULL(totals.bounces, 0) AS bounces,
    IFNULL(totals.timeOnSite, 0) AS time_on_site
  FROM
    \`data-to-insights.ecommerce.web_analytics\`
  WHERE
    totals.newVisits = 1
    AND date BETWEEN '20170501' AND '20170630')
JOIN
  (SELECT
    fullvisitorid,
    IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM
    \`data-to-insights.ecommerce.web_analytics\`
  GROUP BY fullvisitorid)
USING (fullVisitorId)));
"

# 4. Task 3: Build the improved model with feature engineering
echo "Training improved_customer_classification_model (This takes a few minutes)..."
bq query --use_legacy_sql=false "
CREATE OR REPLACE MODEL \`ecommerce.improved_customer_classification_model\`
OPTIONS
  (model_type='logistic_reg', labels = ['will_buy_on_return_visit']) AS
WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
FROM \`data-to-insights.ecommerce.web_analytics\`
GROUP BY fullvisitorid
)
SELECT * EXCEPT(unique_session_id) FROM (
  SELECT
      CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,
      will_buy_on_return_visit,
      MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site,
      IFNULL(totals.pageviews, 0) AS pageviews,
      trafficSource.source,
      trafficSource.medium,
      channelGrouping,
      device.deviceCategory,
      IFNULL(geoNetwork.country, '') AS country
  FROM \`data-to-insights.ecommerce.web_analytics\`,
    UNNEST(hits) AS h
    JOIN all_visitor_stats USING(fullvisitorid)
  WHERE 1=1
    AND totals.newVisits = 1
    AND date BETWEEN '20160801' AND '20170430'
  GROUP BY
  unique_session_id,
  will_buy_on_return_visit,
  bounces,
  time_on_site,
  totals.pageviews,
  trafficSource.source,
  trafficSource.medium,
  channelGrouping,
  device.deviceCategory,
  country
);
"

# Evaluate the improved model
echo "Evaluating improved_customer_classification_model..."
bq query --use_legacy_sql=false "
SELECT roc_auc, accuracy FROM ML.EVALUATE(MODEL \`ecommerce.improved_customer_classification_model\`, (
  WITH all_visitor_stats AS (
  SELECT
    fullvisitorid,
    IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM \`data-to-insights.ecommerce.web_analytics\`
  GROUP BY fullvisitorid
  )
  SELECT * EXCEPT(unique_session_id) FROM (
    SELECT
        CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,
        will_buy_on_return_visit,
        MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,
        IFNULL(totals.bounces, 0) AS bounces,
        IFNULL(totals.timeOnSite, 0) AS time_on_site,
        IFNULL(totals.pageviews, 0) AS pageviews,
        trafficSource.source,
        trafficSource.medium,
        channelGrouping,
        device.deviceCategory,
        IFNULL(geoNetwork.country, '') AS country
    FROM \`data-to-insights.ecommerce.web_analytics\`,
      UNNEST(hits) AS h
      JOIN all_visitor_stats USING(fullvisitorid)
    WHERE 1=1
      AND totals.newVisits = 1
      AND date BETWEEN '20170501' AND '20170630'
    GROUP BY
    unique_session_id,
    will_buy_on_return_visit,
    bounces,
    time_on_site,
    totals.pageviews,
    trafficSource.source,
    trafficSource.medium,
    channelGrouping,
    device.deviceCategory,
    country
  )
));
"

# 5. Task 4: Build finalized model and execute the prediction query for the last month
echo "Training finalized_classification_model (This takes a few minutes)..."
bq query --use_legacy_sql=false "
CREATE OR REPLACE MODEL \`ecommerce.finalized_classification_model\`
OPTIONS
  (model_type='logistic_reg', labels = ['will_buy_on_return_visit']) AS
WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
FROM \`data-to-insights.ecommerce.web_analytics\`
GROUP BY fullvisitorid
)
SELECT * EXCEPT(unique_session_id) FROM (
  SELECT
      CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,
      will_buy_on_return_visit,
      MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site,
      IFNULL(totals.pageviews, 0) AS pageviews,
      trafficSource.source,
      trafficSource.medium,
      channelGrouping,
      device.deviceCategory,
      IFNULL(geoNetwork.country, '') AS country
  FROM \`data-to-insights.ecommerce.web_analytics\`,
    UNNEST(hits) AS h
    JOIN all_visitor_stats USING(fullvisitorid)
  WHERE 1=1
    AND totals.newVisits = 1
    AND date BETWEEN '20160801' AND '20170430'
  GROUP BY
  unique_session_id,
  will_buy_on_return_visit,
  bounces,
  time_on_site,
  totals.pageviews,
  trafficSource.source,
  trafficSource.medium,
  channelGrouping,
  device.deviceCategory,
  country
);
"

echo "Executing PREDICT query for July 2017..."
bq query --use_legacy_sql=false "
SELECT
  *
FROM
  ml.PREDICT(MODEL \`ecommerce.finalized_classification_model\`,
   (
   WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM \`data-to-insights.ecommerce.web_analytics\`
  GROUP BY fullvisitorid
)
  SELECT * EXCEPT(unique_session_id) FROM (
    SELECT
        CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,
        will_buy_on_return_visit,
        MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,
        IFNULL(totals.bounces, 0) AS bounces,
        IFNULL(totals.timeOnSite, 0) AS time_on_site,
        IFNULL(totals.pageviews, 0) AS pageviews,
        trafficSource.source,
        trafficSource.medium,
        channelGrouping,
        device.deviceCategory,
        IFNULL(geoNetwork.country, \"\") AS country
    FROM \`data-to-insights.ecommerce.web_analytics\`,
      UNNEST(hits) AS h
      JOIN all_visitor_stats USING(fullvisitorid)
    WHERE 1=1
      AND totals.newVisits = 1
      AND date BETWEEN \"20170701\" AND \"20170801\" 
    GROUP BY
    unique_session_id,
    will_buy_on_return_visit,
    bounces,
    time_on_site,
    totals.pageviews,
    trafficSource.source,
    trafficSource.medium,
    channelGrouping,
    device.deviceCategory,
    country
  )
  )
)
ORDER BY
  predicted_will_buy_on_return_visit DESC;
"
echo "All tasks complete! You can now check your progress."

```
