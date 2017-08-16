[![NPM version](https://img.shields.io/npm/v/aws-thin-dynamo.svg)](https://www.npmjs.com/package/aws-thin-dynamo)
[![JavaScript Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://standardjs.com)
![Build Status](https://circleci.com/gh/Nike-Inc/aws-thin-dynamo-node.svg?style=shield&circle-token=4281d6b875c7441734baa32820855e45b4b4ae72)


# What is this for?

The [AWS JS SDK](https://github.com/aws/aws-sdk-js) does a lot. For Lambdas is does *too much*; it incurs a 1-2 seconds cold-start time, depending on what you load from it. Even for directly loading the smaller clients, it loads 800kb of code from "core". If you really need to squeeze out that extra performance for Lambda cold-starts, you need a smaller client. This client, with dependencies, is ~30kb. If you are using other thin clients, those dependencies are identical, and share 8kb of the size (its mostly the AWS V4 request signer). In a 256mb Node Lambda the thin client cold starts in ~10ms, compared to ~160ms for the AWS SDK Core.

**aws-thin-dynamo** attempts to be a drop-in replacement for the API it does cover. For ease of use the `callback` parameter can be ommitted from all async calls to get a Promise instead.

As this attempts to duplicate the Official Client's API, you can use [the AWS docs](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html). Don't forget this client will return a promise if you don't provide a `callback` parameter.

# Installation

```
npm i aws-thin-dynamo
```

# Usage

```javascript
var dynamo = require('aws-thin-dynamo')
var client = dynamo({ region: 'us-west-2'})
// client is a API-compliant implemention of the AWS DocumentClient

```

# Constructor differences

The DocumentClient constructor supports some different options than the AWS version. They are below

* **Missing options** - The AWS DocumentClient constructor has these, this client does not
  * `service`. To specify maximum retries or custom backoff timings, see below
* **Additional options** - The AWS Document client does not have these options, this client does
  * `removeEmptyValues` - unlike `convertEmptyValues` which uses the DynamoDB `Null` type, this option will remove empty values entirely. This option has lower precedence than `convertEmptyValues`; if both are specified, this one is ignored.
  * `maxRetries` - Maximum number of retries. **(COMING SOON)**
  * `retryBackoff` - same as the `retryDelayOptions.customBackoff` that the DocumentClient's `service` option takes: a function with the signature `(retryCount) => msToWait`. **(COMING SOON)**

# Automatically Paging Requests

`batchGet` and `batchWrite` allow a maximum of 25 requests, and leave the retrying of "Unprocessed Items" to you. The most common goal with these functions is to retry all unprocessed requests and then move on to the next page, so this client provides two methods that can handle any number of items by breaking them into batches, and automatically retries Unprocessed items.

**batchGetAll** and **batchWriteAll** take the same parameters as **batchGet** and **batchWrite**, except
 * No `callback`, they only support returning promises
 * `params.PageSize` can control the maximum page size used. This number cannot be over 25, but AWS only supports requests of up to 16mb. If 25 items is higher than 16mb you will need to provide a page size small enough that each batch will be under the limit.

 **scanAll** will auto page through the table, it takes the same parameters as **scan** except
 * No `callback`, it only supports returning promises.
 * `ScanLimit` - since `Limit` controls the maximum number of results per scan, `ScanLimit` will stop automatically paging after reaching `ScannedCount` (this is the number of items scanned, not the number of items returned). Since `scan` returns as many requests as it can under `Limit`, it is possible to get back more results than `ScanLimit` if the last scan start under the limit and finishes by crossing it.
 * `ItemLimit - Like `ScanLimit` except it stops are `Count` has been reached.