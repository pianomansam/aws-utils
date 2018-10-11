# AppSync Emulator (Serverless)

This module provides emulation and testing helpers for use with AWS AppSync. Currently this depends on using https://github.com/sid88in/serverless-appsync-plugin and following the conventions.

It is possible to use this emulator without serverless by mirroring the structure defined there. In the future we will provide other methods of configuring the emulator.

## AppSync Features

We aim to support the majority of appsync features (as we use all of them except elastic search).

 - Lambda source (only tested with serverless functions)
 - DynamoDB source (batch operations, all single table operations, etc.)
 - HTTP(S) source
 - NONE source
 - Full VTL support ($util) and compatibility with Java stdlib
 - Support for use with cognito credentials
 - Subscriptions

## Usage

This package will download and run the dynamodb emulator as part of it's appsync emulation features. DynamoDB data is preserved between emulator runs and is stored in `.dynamodb` in the same directory that `package.json` would be in.

### As a CLI

```sh
# NOTE unless you assign a specific port a random one will be chosen.
yarn appsync-emulator --port 62222
```
#### dynamodb with fixed port

optional start dynamodb at a fixed port - e.g. 8000
```sh
# NOTE unless you assign a specific port for dynamodb a random one will be chosen.
yarn appsync-emulator --port 62222 --dynamodb-port 8000
```
to access the dynamodb instance using javascript you need to use the following configuration:
```
const { DynamoDB } = require('aws-sdk');
const dynamodb = new DynamoDB({
  endpoint: 'http://localhost:8000',
  region: 'us-fake-1',
  accessKeyId: 'fake',
  secretAccessKey: 'fake',
});
const client = new DynamoDB.DocumentClient({ service: dynamodb });
```


## Testing

### Jest

We extensively use jest so bundle a jest specific helper (which likely will work for mocha as well).

```js
const gql = require("graphql-tag");
const { AWSAppSyncClient } = require("aws-appsync");

// we export a specific module for testing.
const createAppSync = require("@conduitvc/appsync-emulator-serverless/jest");
// required by apollo-client
global.fetch = require("node-fetch");

describe("graphql", () => {
  const appsync = createAppSync();

  it("Type.resolver", async () => {
    await appsync.client.query({
      query: gql`
        ....
      `
    })
  });
});
```

### generic.

(Below example is jest but any framework will work)

```js
const gql = require("graphql-tag");
const { AWSAppSyncClient } = require("aws-appsync");

// we export a specific module for testing.
const {
  create,
  connect
} = require("@conduitvc/appsync-emulator-serverless/tester");
// required by apollo-client
global.fetch = require("node-fetch");

describe("graphql", () => {
  let server, client;
  beforeEach(async () => {
    server = await create();
    client = connect(
      server,
      AWSAppSyncClient
    );
  });

  // important to clear state.
  afterEach(async () => server.close());
  // very important not to leave java processes lying around.
  afterAll(async () => server.terminate());

  it("Type.resolver", async () => {
    await client.query({
      query: gql`
        ....
      `
    })
  });
});

```


### Lambda <> DynamoDB

If you need your lambda functions to interact with the local dynamoDB emulator:

```js
const { DynamoDB } = require('aws-sdk');

const dynamodb = new DynamoDB({
  endpoint: process.env.DYNAMODB_ENDPOINT,
  region: 'us-fake-1',
  accessKeyId: 'fake',
  secretAccessKey: 'fake',
});
const client = new DynamoDB.DocumentClient({ service: dynamodb });

module.exports.myFn = async (event, context, callback) => {
  const TableName = process.env[`DYNAMODB_TABLE_${YOURTABLENAME}`];
  const { id } = event.arguments;
  const dynamoResult = await client
  .get({
    TableName,
    Key: { id },
  }).promise();
  
  return dynamoResult;
}
```
