---
title: Troubleshoot issues writing data
seotitle: Troubleshoot issues writing data to InfluxDB
weight: 106
description: >
  Troubleshoot issues writing data.
  Find response codes for failed writes.
  Discover how writes fail, from exceeding rate or payload limits, to syntax
  errors and schema conflicts.
menu:
  influxdb3_clustered:
    name: Troubleshoot issues
    parent: Write data
influxdb3/clustered/tags: [write, line protocol, errors]
related:
  - /influxdb3/clustered/reference/syntax/line-protocol/
  - /influxdb3/clustered/write-data/best-practices/
  - /influxdb3/clustered/reference/internals/durability/
---

Learn how to avoid unexpected results and recover from errors when writing to
{{% product-name %}}.

- [Handle write responses](#handle-write-responses)
  - [Review HTTP status codes](#review-http-status-codes)
- [Troubleshoot failures](#troubleshoot-failures)
- [Troubleshoot rejected points](#troubleshoot-rejected-points)

## Handle write responses

{{% product-name %}} does the following when you send a write request:

1.  Validates the request.
2.  If successful, attempts to ingest data from the request body; otherwise,
    responds with an [error status](#review-http-status-codes).
3.  Ingests or rejects data in the batch and returns one of the following HTTP
    status codes:

    - `204 No Content`: All data in the batch is ingested.
    - `400 Bad Request`: Some or all of the data has been rejected.
      Data that has not been rejected is ingested and queryable.

The response body contains error details about
[rejected points](#troubleshoot-rejected-points), up to 100 points.

Writes are synchronous--the response status indicates the final status of the
write and all ingested data is queryable.

To ensure that InfluxDB handles writes in the order you request them,
wait for the response before you send the next request.

### Review HTTP status codes

InfluxDB uses conventional HTTP status codes to indicate the success or failure of a request.
The `message` property of the response body may contain additional details about the error.
Write requests return the following status codes:

| HTTP response code              | Message                                                                 | Description    |
| :-------------------------------| :---------------------------------------------------------------        | :------------- |
| `204 "Success"`                 |                                                                         | If InfluxDB ingested the data |
| `400 "Bad request"`             | error details about rejected points, up to 100 points: `line` contains the first rejected line, `message` describes rejections | If some or all request data isn't allowed (for example, if it is malformed or falls outside of the bucket's retention period)--the response body indicates whether a partial write has occurred or if all data has been rejected |
| `401 "Unauthorized"`            |                                                                         | If the `Authorization` header is missing or malformed or if the [token](/influxdb3/clustered/admin/tokens/) doesn't have [permission](/influxdb3/clustered/reference/cli/influxctl/token/create/#examples) to write to the database. See [examples using credentials](/influxdb3/clustered/get-started/write/#write-line-protocol-to-influxdb) in write requests. |
| `404 "Not found"`               | requested **resource type** (for example, "organization" or "database"), and **resource name**     | If a requested resource (for example, organization or database) wasn't found |
| `500 "Internal server error"`   |                                                                         | Default status for an error |
| `503` "Service unavailable"     |                                                                         | If the server is temporarily unavailable to accept writes. The `Retry-After` header describes when to try the write again.

If your data did not write to the database, see how to [troubleshoot rejected points](#troubleshoot-rejected-points).

## Troubleshoot failures

If you notice data is missing in your database, do the following:

- Check the `message` property in the response body for details about the error.
- If the `message` describes a field error, [troubleshoot rejected points](#troubleshoot-rejected-points).
- Verify all lines contain valid syntax ([line protocol](/influxdb3/clustered/reference/syntax/line-protocol/)).
- Verify the timestamps in your data match the [precision parameter](/influxdb3/clustered/reference/glossary/#precision) in your request.
- Minimize payload size and network errors by [optimizing writes](/influxdb3/clustered/write-data/best-practices/optimize-writes/).

## Troubleshoot rejected points

InfluxDB rejects points that fall within the same partition (default partitioning
is by measurement and day) as existing bucket data and have a different data type
for an existing field.

Check for [field data type](/influxdb3/clustered/reference/syntax/line-protocol/#data-types-and-format)
differences between the rejected data point and points within the same database
and partition--for example, did you attempt to write `string` data to an `int` field?
