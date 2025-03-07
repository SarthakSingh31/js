---
title: Convert AWS Lambda Functions to Fermyon Spin
---

This pattern converts a serverless function to a spin function designed to run on [Fermyon](https://www.fermyon.com/).

tags: #js, #migration, #serverless, #fermyon, #alpha

```grit
engine marzano(0.1)
language js

predicate insert_statement($statement) {
    $program <: or {
        contains `export $_` as $old => `$statement\n\n$old`,
        contains `"use strict"` as $old => `$old\n\n$statement`,
        $program => js"$statement\n\n$program"
    }
}

pattern spin_fix_response() {
     or {
         object($properties) where {
            $properties <: contains bubble {
                pair($key, $value) where {
                    $key <: "body",
                    $value <: $old => js"encoder.encode($old).buffer"
                }
            },
            $properties <: contains bubble {
                pair($key, $value) where {
                    $key <: "statusCode",
                    $key => js"status"
                }
            },
        },
        object() as $obj where {
          $obj => js"{
            status: 200,
            body: encoder.encode($obj).buffer
          }"
        }
    }
}

pattern spin_fix_request($event) {
    `$event.request.$prop` => `JSON.parse(decoder.decode($event.body)).$prop` where {
        insert_statement(statement=js"const decoder = new TextDecoder('utf-8')")
    }
}

predicate spin_uses_ts() {
    $program <: contains type_annotation()
}

pattern spin_main_fix_handler() {
  or {
      js"module.exports.$_ = ($args) => { $body }",
      js"export const $_ = ($args) => {$body }"
    } as $func where {
        $request = `request`,
        $args <: or { [$event_arg], [$event_arg, $context, $callback] },
        $event_arg <: contains identifier() as $event,
        $body <: maybe contains $event => `request`,
        $body <: contains or {
            `return $response`,
            `callback(null, $response)` => `return $response`
        } where {
            $response <: or {
                spin_fix_response(),
                identifier() where { $body <: contains `$response = $def` where $def <: spin_fix_response() }
            }
        },
        $event => `request`,
        if (spin_uses_ts()) {
            $req_type = `HttpRequest`,
            $req_type <: ensure_import_from(source=`"@fermyon/spin-sdk"`),
            $res_type = `HttpResponse`,
            $res_type <: ensure_import_from(source=`"@fermyon/spin-sdk"`),
            $new = js"export async function handleRequest($event: $req_type): Promise<$res_type> {
        $body
    }",
        } else {
            $new = js"export async function handleRequest($event) {
        $body
    }",
        }
    } => $new
}

pattern spin_remove_lambda() {
    `import $_ from "aws-lambda"` => .
}

pattern spin_main_fix_request() {
    `function handleRequest($request) { $body }` where {
        $body <: contains spin_fix_request(event=`request`)
    }
}

pattern spin_add_encoder() {
    `encoder.encode` where {
        insert_statement(statement=js"const encoder = new TextEncoder('utf-8');")
    }
}

sequential {
    contains spin_main_fix_handler(),
    maybe contains spin_main_fix_request(),
    maybe contains spin_add_encoder(),
    maybe contains spin_remove_lambda(),
}
```

## Converts a basic Serverless component

```js
module.exports.handler = async (event) => {
  return {
    statusCode: 200,
    body: JSON.stringify(
      {
        message: 'Go Serverless v3.0! Your function executed successfully!',
        input: event,
      },
      null,
      2,
    ),
  };
};
```

```js
const encoder = new TextEncoder('utf-8');

export async function handleRequest(request) {
  return {
    status: 200,
    body: encoder.encode(
      JSON.stringify(
        {
          message: 'Go Serverless v3.0! Your function executed successfully!',
          input: request,
        },
        null,
        2,
      ),
    ).buffer,
  };
}
```

## Converts a TypeScript handler

```ts
import { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';

export const hello = async (event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> => {
  return {
    statusCode: 200,
    body: JSON.stringify(
      {
        message: 'Go Serverless v3.0! Your function executed successfully!',
        input: event,
      },
      null,
      2,
    ),
  };
};
```

```ts
import { HttpRequest, HttpResponse } from '@fermyon/spin-sdk';

const encoder = new TextEncoder('utf-8');

export async function handleRequest(request: HttpRequest): Promise<HttpResponse> {
  return {
    status: 200,
    body: encoder.encode(
      JSON.stringify(
        {
          message: 'Go Serverless v3.0! Your function executed successfully!',
          input: request,
        },
        null,
        2,
      ),
    ).buffer,
  };
}
```

## Converts a handler with inputs

This example is based on [serverless example](https://github.com/custodian-sample-org/serverless-examples/blob/v3/aws-node-alexa-skill/handler.js).

```js
'use strict';

// Returns a random integer between min (inclusive) and max (inclusive)
const getRandomInt = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;

module.exports.luckyNumber = (event, context, callback) => {
  const upperLimit = event.request.intent.slots.UpperLimit.value || 100;
  const number = getRandomInt(0, upperLimit);
  const response = {
    version: '1.0',
    response: {
      outputSpeech: {
        type: 'PlainText',
        text: `Your lucky number is ${number}`,
      },
      shouldEndSession: false,
    },
  };

  callback(null, response);
};
```

```js
'use strict';

// Returns a random integer between min (inclusive) and max (inclusive)
const getRandomInt = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;

const decoder = new TextDecoder('utf-8');

const encoder = new TextEncoder('utf-8');

export async function handleRequest(request) {
  const upperLimit = JSON.parse(decoder.decode(request.body)).intent.slots.UpperLimit.value || 100;
  const number = getRandomInt(0, upperLimit);
  const response = {
    status: 200,
    body: encoder.encode({
      version: '1.0',
      response: {
        outputSpeech: {
          type: 'PlainText',
          text: `Your lucky number is ${number}`,
        },
        shouldEndSession: false,
      },
    }).buffer,
  };

  return response;
}
```
