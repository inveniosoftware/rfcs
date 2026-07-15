- Start Date: 2025-08-01
- RFC PR: [#104](https://github.com/inveniosoftware/rfcs/pull/104)
- Authors: Jakob Miesner

# Library KPIs

## Summary
Add endpoints to InvenioILS to retrieve KPIs about the performance of the library.

## Motivation
The CERN Library would like to have two dashboards displaying its KPIs for two different user groups:
 
1. Internal Dashboard
This dashboard is used internally by the librarians to measure the performance of certain processes.
E.g. when a patron requests a document, how much time passes between the patron issuing the request and the patron receiving the book.
It allows the librarians to react to find bottlenecks and track the evolution of the library's performance over time. 

2. Outsider/Stakeholder Dashboard
The Audience of this Dashboard is not the librarians, but rather the patrons and management.
It displays simpler KPIs that show if the library is working well, while being less detailed and technical than the internal dashboard.


## Detailed design
This RFC proposes to add endpoints to `invenio-app-ils` that provide the data required for the KPIs requested by the CERN Library.
Displaying the KPIs in dashboards is out of scope for this RFC.
In this section we will describe how the KPIs will be implemented in `invenio-app-ils`.
Hereby some KPIs are split up into individual queries, which can then be combined in the dashboard to compute the final KPI value.
Splitting up the requested KPIs into individual queries allows for more flexibility in how the KPIs are later displayed.
The section will also describe how the dashboard can request those queries from the API.

Some queries are already possible with the existing Invenio ILS API, while others require new implementations.

### Usage of `invenio-stats`
The package `invenio-stats`, which is already used by `invenio-app-ils`, will be used to implement some of the KPIs.
Certain stats are fully implemented through `invenio-stats` (e.g. [nr. 4](#4-number-of-changes-to-the-library-collections)). 
Others only use it to store additional information for the records, which is then later added to the records index (e.g. [nr. 3](#3-availability-of-requested-documents)).

### Histogram-based KPIs
We also add a new way to request time series data from the API for certain record types.
This is done by adding a new endpoint for them called a `histogram` endpoint, which is available under `/api/<record-type>/stats`.
The endpoint allows to specify fields to group by and metrics to compute for those groups, which wraps the functionality of the search engine, while sanitizing the input and output.
This allows to flexibly query different KPIs from the same endpoint.
The histogram will be implemented in a reusable fashion and exposed for multiple record types.

#### Example usage of the histogram endpoint
If one for example wants to see how the distribution of loan delivery methods over time (e.g. start date of loans, monthly) looks like, one can request:
`group_by=[{"field": "start_date", "interval": "1M"} , {"field": "delivery.method"}]`
If one would additionally like to know the average duration of those loans, one can add a metric:
`metrics=[{"field": "extra_data.stats.loan_duration", "aggregation": "avg"} `


### The specific KPIs are implemented as follows
We describe below how each of the requested KPIs will be implemented and how they can be queried from the API.

#### 1. Turnover rate of the Library collection:
##### 1. number of new loans / number of loanable items
**NOTE**: this KPI is described in ISO 11620:2023 A.2.1.1

- query - number of new loans:
can be queried from the api by requesting e.g. `/api/circulation/loans?q=_created:[2025-01-01 TO 2025-12-31]`
  result is in `hits.total`
can be queried using the histogram at `/api/circulation/stats/loans?group_by=[{"field": "_created", "interval": "1M"}]`
  results are in `buckets[x].doc_count`

- query - number of loanable items:
**NOTE**: no time series data will be available for this query, as the number of loanable items is only available at the time of the request. Thus the dashboard has to store the historical data itself if needed.
Nothing to be implemented here, as it is already possible.
can be queried from the api by requesting `/api/items/`
  result is in `aggregations.status.buckets.first(x => x.key == "CAN_CIRCULATE").doc_count` 


##### 2. number of renewals / number of loanable items
- query - number of renewals: 
Implemented with `invenio-stats`.
We listen to existing signal `invenio_circulation.signals.loan_state_changed` and create event for all transitions.
Can be queried by requesting `/api/stats` with the following body:

```
{
  "extensions": {
    "stat": "loan-transitions",
    "params": {
      "start_date" :"2023-01-12",
      "end_date" :"2026-11-30",
      "trigger": "extend"
    }
  }
}
```
result is in `extensions.buckets[x].count`

- query - number of loanable items:
  see [1.1: "query - number of loanable items"](#1-number-of-new-loans--number-of-loanable-items)


#### 2. Average duration of loan:
Add a new field to the loan index called `loan_duration` under `extra_data.stats`, which is computed as the difference between the `end_date` and `start_date` of the loan in days when the loan is closed.

- query: average loan duration 
can be queried using the histogram at `/api/circulation/loans/stats?group_by=[{"field": "start_date", "interval": "1M"}]&metrics=[{"field": "extra_data.stats.loan_duration", "aggregation": "avg"}]`
  results are in `buckets[x].metrics.avg__extra_data.stats.loan_duration`



#### 3. Availability of requested documents:
It was requested to differentiate between loans that were requested when items were available vs. not available.
Thus, we add a new field to the loan index called `available_items_during_request`, which is a boolean flag indicating whether there were available items for the requested document at the time of the request.
As this information is not directly required for normal loan operations, we will use `invenio-stats` to store this information when the loan request is created and then later add it to the loan index.
For this purpose the `loan-transitions` event created in [1.2: "query - number of renewals"](#2-number-of-renewals--number-of-loanable-items) will be extended to also store the number of available physical items for the requested document when the trigger is `request`.
The loan indexer will then read this information from the `invenio-stats` `events index and add it to the loan index.


##### 1. Loans with available items vs. without available items vs. self checkout
- query: number of loan requests with available items vs. without available items vs. self checkout
non self-checkout requests can be queried using the histogram at `/api/circulation/loans/stats?group_by=[{"field": "_created", "interval": "1M"}, {"field": "extra_data.stats.available_items_during_request"}]&q=!delivery.method: SELF-CHECKOUT`
self checkout requests can be queried using the histogram at `/api/circulation/loans/stats?group_by=[{"field": "_created", "interval": "1M"}]&q=delivery.method: SELF-CHECKOUT`
  results are in `buckets[x].doc_count`

##### 2. Average Waiting Time := waiting time between loan (request) creation and loan start time / number of loans started
- query: average waiting time between loan (request) creation and loan start time
can be queried using the histogram at `/api/circulation/loans/stats?q=start_date:*&metrics=[{"field": "extra_data.stats.waiting_time", "aggregation": "avg"}]&group_by=[{"field": "_created", "interval": "1M"}, {"field": "extra_data.stats.available_items_during_request"}]`
results are in `buckets[x].metrics.avg__extra_data.stats.waiting_time`


#### 4. Number of changes to the Library collections:
- query: number of creations, updates, deletions of all ILS records
The following record types were directly requested by the CERN Library, but we will track all record types:documents, physical items, e-items, loans
Implemented with `invenio-stats`.
Listen to existing signals from `invenio_records`: `after_record_insert`, `after_record_update`, `after_record_delete`
Aggregate daily over composite field `aggregation_id` which is composed of:
    - `pid_type` := record pid type (e.g. `docid`, `eitmid`, ...)
    - `method` := C(R)UD method used to modify the record (`create`, `update`, `delete`)
    - optionally: `user_id` but only for librarians who made the change.
      - Only librarians for GDPR reasons.
    - example values of the field would be: `docid__create`, `eitmid__update__2`, ...

#### 5. Patrons:
##### 1. Number of unique patrons with an activity on their account for a given period (logging in counts as active)
- query: count distinct patron ids that logged in
Use existing signal `invenio_accounts.views.LoginView::login_user`.
Currently `invenio-stats` always requires a field to aggregate over and filter by.
Thus, a new aggregator for `invenio-stats` needs to be added that can aggregate without grouping by a field.
Can be queried by requesting `/api/stats` with the following body:
```
{
  "distinct-patron-logins": {
    "stat": "distinct-patron-logins",
    "params": {
      "start_date" :"2023-01-12",
      "end_date" :"2026-11-30"
    }
  }
}
```

##### 2. Loan issued for a period associated with Patron's department
- query: number of loans issued per department
Can be queried using the histogram at `/api/circulation/loans/stats?group_by=[{"field": "patron.department.keyword"}]`
  results are in `buckets[x].doc_count`

#### 6. Overdue loans;
- query: percentage of overdue loans (active overdue loans / active loans)
**NOTE**: no time series data will be available for this query, as the number of active overdue loans is only available at the time of the request. Thus the dashboard has to store the historical data itself if needed. 
can be queried from the api by requesting `/api/circulation/loans/`
  result is in `aggregations.returns.end_date.buckets.first(x => x.key == "Overdue").doc_count` and `aggregations.state.buckets.first(x => x.key == "ITEM_ON_LOAN").doc_count`


#### 7. Purchase orders:
##### 1. Waiting time: Received time of the Purchase Order - Related Literature Request creation time
Add a new field to the acquisition orders index called `document_request_waiting_time` under `stats`, which is computed as the `received_date` of the purchase order minus the `created` date of the related literature request (if any) in days.

- query: average waiting time between literature request creation and purchase order received status
`/api/acquisition/stats?group_by=[{"field": "_created", "interval": 1M}]&metrics=[{"field": "stats.document_request_waiting_time", "aggregation": "avg"}]`


##### 2. Ordering time: Received time of the Purchase Order - Purchase Order creation time
Add a new field to the acquisition orders index called `order_processing_time` under `stats`, which is computed as the `received_date` minus the `created` date of the purchase order in days.

- query: average ordering time between purchase order creation and received status
`/api/acquisition/stats?group_by=[{"field": "_created", "interval": 1M}]&metrics=[{"field": "stats.order_processing_time", "aggregation": "avg"}]`


#### 8. Literature requests:
##### 1. Efficiency: Time difference between Purchase Order/Borrowing Request - Literature request creation time
- query: time difference between Purchase Order/Borrowing Request - Literature request creation time
we add the field `provider_creation_delay` under `stats` to the document request index, which is computed as the difference between the `created` date of the literature literature and the `created` date of the first associated purchase order or borrowing request in days.
can be queried using the histogram at `/api/document-requests/stats?group_by=[{"field": "_created", "interval": "1M"}]&metrics=[{"field": "stats.provider_creation_delay", "aggregation": "avg"}]`
  results are in `buckets[x].metrics.avg__.stats.provider_creation_delay`

##### 2.  Fulfillment: percentage of accepted vs. declined literature requests (differentiate decline reason 'available in catalogue' from other reasons).
**NOTE**: no time series data will be available for this query, as the number of loanable items is only available at the time of the request. Thus the dashboard has to store the historical data itself if needed. 
- query : number of accepted vs. declined literature requests 
can be queried with `/api/document-requests`
  - result is in `aggregations.state.buckets.first(x => x.key == "ACCEPTED").doc_count` and `aggregations.state.buckets.first(x => x.key == "DECLINED").doc_count`
  - results for decline reason are in `aggregations.decline_reason.buckets`

## Alternatives

### Using invenio-stats for all KPIs
One alternative considers was to implement all KPIs using `invenio-stats`.
This would allow us to not extend the indexes of the records and only rely on the event schema of `invenio-stats`.
However, some KPIs require information that does not fit into the schema of events.
For example the duration of loans requires storing both the start and end date of the loan in the event and then aggregating over the difference of those two dates.
This would lead to a lot of complexity in the event generation and aggregation code.
Thus, we decided to only use `invenio-stats` for KPIs where it makes sense and use the histogram endpoint and extending the index for the others.

### Not indexing computable fields like loan `loan_duration`
Another alternative considered would be to not index computable fields like `loan_duration` in the records index, but rather compute them on the fly when requested.
While this would reduce the number of fields in the index, it would lead to a lot of complexity in the API code and would require more computation in the search engine when requesting.

### KPI 2 and 3.2 - Alternative approach to measure waiting time and loan duration
An alternative approach to measure the waiting time and loan duration would be do count the loans that are respectively waiting or active at the end of each day.
This would give a more direct measure of how many loans are active or waiting over time.
However, this does not allow to compute an average, as it is unsure what divisor to use.

### Storing the extra data in the records 
For loans we had to store additional data `available_items_during_request` for loans.
We solved this by utilizing invenio-stats to store extra data on certain events (like the number of available items during the creation of a loan request).
An alternative considered was to store this extra data directly in the loan record.
However, this would lead to data in the loan record that is irrelevant for most use cases and only needed for the KPIs.