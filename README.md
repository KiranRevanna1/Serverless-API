# Building a serverless API
## Add an API to Create a Note
We’ll first add an API to create a note. This API will take the note object as the input and store it in the database with a new id. The note object will contain the `content` field (the content of the note) and an `attachment` field (the URL to the uploaded file).

### Creating a Stack
 Create a new file in `stacks/ApiStack.js` and add the following.
```sh
import { Api, use } from "@serverless-stack/resources";
import { StorageStack } from "./StorageStack";

export function ApiStack({ stack, app }) {
  const { table } = use(StorageStack);

  // Create the API
  const api = new Api(stack, "Api", {
    defaults: {
      function: {
        permissions: [table],
        environment: {
          TABLE_NAME: table.tableName,
        },
      },
    },
    routes: {
      "POST /notes": "functions/create.main",
    },
  });

  // Show the API endpoint in the output
  stack.addOutputs({
    ApiEndpoint: api.url,
  });

  // Return the API resource
  return {
    api,
  };
}
```
We are doing a couple of things of note here.
<ul>
<li>We are creating a new stack for our API. We could’ve used the stack we had previously created for DynamoDB and S3. But this is a good way to talk about how to share resources between stacks.</li>

<li>This new `ApiStack` references the `table` resource from the `StorageStack` that we created previously.</li>

<li>We are creating an API using SST’s `Api` construct.</li>

<li>We are passing in the name of our DynamoDB table as an environment variable called `TABLE_NAME`. We’ll need this to query our table.</li>

<li>The first route we are adding to our API is the `POST /notes` route. It’ll be used to create a note.</li>

<li>We are giving our API permission to access our DynamoDB table by setting `permissions: [table]`.</li>

<li>Finally, we are printing out the URL of our API as an output by calling `stack.addOutputs`. We are also exposing the API publicly so we can refer to it in other stacks.</li>
</ul>
### Adding to the App
Let’s add this new stack to the rest of our app.

 In `stacks/index.js`, import the API stack at the top.
```sh
import { ApiStack } from "./ApiStack";
```
 And, replace the main function with -
```sh
export default function main(app) {
  app.setDefaultFunctionProps({
    runtime: "nodejs16.x",
    srcPath: "services",
    bundle: {
      format: "esm",
    },
  });
  app.stack(StorageStack).stack(ApiStack);
}
```
### Add the Function
Now let’s add the function that’ll be creating our note.

 Create a new file in s`ervices/functions/create.js` with the following.
```sh
import * as uuid from "uuid";
import AWS from "aws-sdk";

const dynamoDb = new AWS.DynamoDB.DocumentClient();

export async function main(event) {
  // Request body is passed in as a JSON encoded string in 'event.body'
  const data = JSON.parse(event.body);

  const params = {
    TableName: process.env.TABLE_NAME,
    Item: {
      // The attributes of the item to be created
      userId: "123", // The id of the author
      noteId: uuid.v1(), // A unique uuid
      content: data.content, // Parsed from request body
      attachment: data.attachment, // Parsed from request body
      createdAt: Date.now(), // Current Unix timestamp
    },
  };

  try {
    await dynamoDb.put(params).promise();

    return {
      statusCode: 200,
      body: JSON.stringify(params.Item),
    };
  } catch (e) {
    return {
      statusCode: 500,
      body: JSON.stringify({ error: e.message }),
    };
  }
}
```
There are some helpful comments in the code but let’s go over them quickly.
<ul>
<li>Parse the input from the `event.body`. This represents the HTTP request body.
<li>It contains the contents of the note, as a string — `content`.
<li>It also contains an `attachment`, if one exists. It’s the filename of a file that will be uploaded to our S3 bucket.
<li>We read the name of our DynamoDB table from the environment variable using p`rocess.env.TABLE_NAME`. You’ll recall that we set this above while configuring our API.
<li>The userId is the id for the author of the note. For now we are hardcoding it to `123`. Later we’ll be setting this based on the authenticated user.
<li>Make a call to DynamoDB to put a new object with a generated `noteId` and the current date as the `createdAt`.
<li>And if the DynamoDB call fails then return an error with the HTTP status code `500`.
Let’s go ahead and install the npm packages that we are using here.

 Run the following in the `services/` folder.
```sh
$ npm install aws-sdk uuid
```
<li><b>aws-sdk</b> allows us to talk to the various AWS services.
<li><b>uuid</b>generates unique ids.</li>
</ul>

## Deploy Our Change
If you switch over to your terminal, you’ll notice that you are being prompted to redeploy your changes. Go ahead and hit ENTER.

Note that, you’ll need to have `sst start` running for this to happen. If you had previously stopped it, then running `npx sst start` will deploy your changes again.

You should see that the new API stack has been deployed.
```sh
Stack dev-notes-ApiStack
  Status: deployed
  Outputs:
    ApiEndpoint: https://5bv7x0iuga.execute-api.us-east-1.amazonaws.com
 ```
It includes the API endpoint that we created.

## Test the API
Now we are ready to test our new API.

Head over to the API tab in the SST Console and check out the new API.

<img src="https://imgur.com/7zaUCSm.png" alt="Homepage view 3" width=1000 height=500/>

Here we can test our APIs.

 Add the following request body to the Body field and hit Send.
```sh
{"content":"Hello World","attachment":"hello.jpg"}
```
You should see the create note API request being made.
<img src="https://imgur.com/ZsJJIga.png" alt="Homepage view 3" width=1000 height=500/>


Here we are making a POST request to our create note API. We are passing in the content and attachment as a JSON string. In this case the attachment is a made up file name. We haven’t uploaded anything to S3 yet.

If you head over to the DynamoDB tab, you’ll see the new note.

<img src="https://imgur.com/UgwlwWH.png" alt="Homepage view 3" width=1000 height=500/>

Make a note of the `noteId`. We are going to use this newly created note in the next chapter.

## Refactor Our Code
Before we move on to the next chapter, let’s quickly refactor the code since we are going to be doing much of the same for all of our APIs.

 Start by replacing our `create.js` with the following.
```sh
import * as uuid from "uuid";
import handler from "../util/handler";
import dynamoDb from "../util/dynamodb";

export const main = handler(async (event) => {
  const data = JSON.parse(event.body);
  const params = {
    TableName: process.env.TABLE_NAME,
    Item: {
      // The attributes of the item to be created
      userId: "123", // The id of the author
      noteId: uuid.v1(), // A unique uuid
      content: data.content, // Parsed from request body
      attachment: data.attachment, // Parsed from request body
      createdAt: Date.now(), // Current Unix timestamp
    },
  };

  await dynamoDb.put(params);

  return params.Item;
});
```
This code doesn’t work just yet but it shows you what we want to accomplish:

<li>We want to make our Lambda function async, and simply return the results.
<li>We want to simplify how we make calls to DynamoDB. We don’t want to have to create a new AWS.DynamoDB.DocumentClient().
<li>We want to centrally handle any errors in our Lambda functions.
Finally, since all of our Lambda functions will be handling API endpoints, we want to handle our HTTP responses in one place.
Let’s start by creating the `dynamodb` util.

 From the project root run the following to create a `services/util` directory.
```sh
$ mkdir services/util
```
 Create a `services/util/dynamodb.js` file with:
```sh
import AWS from "aws-sdk";

const client = new AWS.DynamoDB.DocumentClient();

export default {
  get: (params) => client.get(params).promise(),
  put: (params) => client.put(params).promise(),
  query: (params) => client.query(params).promise(),
  update: (params) => client.update(params).promise(),
  delete: (params) => client.delete(params).promise(),
};
```
Here we are creating a convenience object that exposes the DynamoDB client methods that we are going to need in this guide.

 Also create a `services/util/handler.js` file with the following.
```sh
export default function handler(lambda) {
  return async function (event, context) {
    let body, statusCode;

    try {
      // Run the Lambda
      body = await lambda(event, context);
      statusCode = 200;
    } catch (e) {
      console.error(e);
      body = { error: e.message };
      statusCode = 500;
    }

    // Return HTTP response
    return {
      statusCode,
      body: JSON.stringify(body),
    };
  };
}
```
Let’s go over this in detail.

<li>We are creating a handler function that we’ll use as a wrapper around our Lambda functions.
<li>It takes our Lambda function as the argument.
<li>We then run the Lambda function in a try/catch block.
<li>On success, we JSON.stringify the result and return it with a 200 status code.
<li>If there is an error then we return the error message with a 500 status code.
It’s important to note that the handler.js needs to be imported before we import anything else. This is because we’ll be adding some error handling to it later that needs to be initialized when our Lambda function is first invoked.

Next, we are going to add the API to get a note given its id.

### Common Issues

<li>Response statusCode: 500</li>

If you see a `statusCode: 500` response when you invoke your function, the error has been reported by our code in the `catch` block. You’ll see a `console.error` is included in our `util/handler.js` code above. Incorporating logs like these can help give you insight on issues and how to resolve them.
```sh
} catch (e) {
  // Prints the full error
  console.error(e);

  body = { error: e.message };
  statusCode = 500;
}

```
