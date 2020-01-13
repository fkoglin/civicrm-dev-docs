# API v4 Changelog

This page lists additions to the v4 api with each new release of CiviCRM Core.

Also see: [Differences Between Api v3 and v4](/api/v4/differences-with-v3.md).

### 5.23 Added PaymentProcessor and PaymentProcessorType APIv4 Entities

See https://github.com/civicrm/civicrm-core/pull/15624

### 5.23 $index param supports array input

CiviCRM 5.23 supports two new modes for the `$index` param - associative and non-associative array. See https://github.com/civicrm/civicrm-core/pull/16257

### 5.23 Converts field values to correct data type

The api historically returns everything as a raw string from the query instead of converting it to the correct variable type (bool, int, float). As of CiviCRM 5.23 this is fixed for all DAO-based entities. See https://github.com/civicrm/civicrm-core/pull/16274
