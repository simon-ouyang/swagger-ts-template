# swagger-ts-template

This fork's npm package:

```
yarn add @proerd/swagger-ts-template
```

Quick 'n dirty solution to integrate swagger v2 into a typescript codebase. Partial OAI v3 support.

- Generates an (opinionated) typescript consumer API. Call REST APIs just as functions. Forget about HTTP details. Get doc suggestions and typed responses.

```javascript
var generator = require("swagger-ts-template")
var swaggerFile = require("./api.json")

generator.genPaths(swaggerFile, { output: "./api" }).then(() => console.log("okay"))
```

## What's different on this one?

This has been serving me great for a couple of years.

- The scaffold gives almost no opinion on how should you do your requests. Receive the processed
  playload, do the request with whatever fetch or axios or any other thing, return a response or throw an error;
- The default "payload processing" is also easily extensible;
- You can add extra input or output arguments through interface augmenting, which works fine for covering special cases;

([detailed article with motivations](https://dev.to/wkrueger/integrating-apis-to-a-typescript-frontend-with-openapi-swagger-3521))

## genPaths options

Generates a typed API consumer on a defined folder. `genTypes` is already included.

```ts
export async function genPaths(swaggerDoc: SwaggerDoc, opts: genPathsOpts): Promise<void>

type genPathsOpts = {
  output: string //output folder path
  moduleStyle: "commonjs" | "esm"
  failOnMissingOperationId?: boolean
  typesOpts?: genTypesOpts
  mapOperation?: (
    operation: Operation,
    pathItem: SwaggerIo.V2.SchemaJson.Definitions.PathItem,
    pathKey: string,
    methodKey: string
  ) => Operation
  templateString?: string
}
```

## Handling incomplete APIs

`swagger-ts-template` relies on the presence of `operationId` and `tags` on the swagger definition.

If those are missing, it will attempt to generate default values from the path names.

The default generated values may not fit an API. In that case, you can complement a swagger definition by adding yourelf those fields though `mapOperation`.

## Setting up the consumer API

You have to bootstrap the api skeleton telling how should you run the requests.

This setting is global and must be run before the 1st request takes place.

Sample:

```typescript
import { SwaggerRequester, IRequest, IOperation, settings } from "./gen/api-common"

const BACKEND_URL = process.env.GEOS_API_URL!

class RestRequester extends SwaggerRequester {
  async handler(
    request: IRequest & GApiCommon.MergeToRequest,
    input: Record<string, any>,
    operation: IOperation
  ) {
    const url = new URL(BACKEND_URL + request.url)
    const params = request.query || {}
    Object.keys(params).forEach(key => url.searchParams.append(key, params[key]))
    const body = ["GET", "DELETE"].includes(request.verb!)
      ? undefined
      : JSON.stringify(request.body)
    const fetchResp = await fetch(url.toString(), {
      method: request.verb,
      body,
      headers: {
        "Content-Type": "application/json"
      }
    })
    if (fetchResp.status === 204) return {}
    if (String(fetchResp.status).charAt(0) !== "2") {
      const clone = fetchResp.clone()
      try {
        const json = await fetchResp.json()
        const msg = json?.error?.message
        const err = Error(msg || "Request error.")
        ;(err as any).pass = true
        throw err
      } catch (err) {
        if (err.pass) throw err
        const txt = await clone.text()
        throw Error(txt)
      }
    }
    const out = fetchResp.json()
    return out
  }
}

const requester = new RestRequester()
settings.getRequester = () => requester
```

## Generated consumer API

- All declared types are exported as `interface`'s or type aliases when necessary;
- "Anonymous" types (actually, the request and response types)
  will also get an interface for them, with a generated name (if they are not already present in "declarations");
- Methods are grouped in files by their tag and named by their operationId;
- The transport is abstracted away. All inputs are passed by their names, there is no distiction whether they are in the path, query parameters or body; The method declarations carry the necessary metadata to achieve this.

Sample generated file excerpt:

```ts
import * as Types from "../api-types"
import * as ApiCommon from "../api-common"

export type getPriceEstimates_Type = {
  end_latitude: number
  end_longitude: number
  start_latitude: number
  start_longitude: number
}
export const getPriceEstimates = ApiCommon.requestMaker<
  getPriceEstimates_Type,
  Types.getPriceEstimates_Response
>({
  id: "getPriceEstimates",
  path: "/estimates/price",
  verb: "GET",
  parameters: [
    { name: "end_latitude", required: true, in: "query" },
    { name: "end_longitude", required: true, in: "query" },
    { name: "start_latitude", required: true, in: "query" },
    { name: "start_longitude", required: true, in: "query" }
  ]
})

///...
```

Sample consumer invocation:

```ts
import CustomerApi = require("./api/modules/Customer")
let customer = await CustomerApi.getCustomer({
  customerId: 999
})
```

See also the `samples` path in this repo.

## Personalization

- The input parameters from `genPaths` may be used to tweak the generation a bit; Most notably, you can remap the operation objects to do things such as renaming methods;
- You can extend the request and response types by augmenting either
  `GApiCommon#MergeToRequest` or `GApiCommon#MergeToResponse` global interfaces.
- You can extend the source classes to override something...

```ts
declare global {
  namespace GApiCommon {
    interface MergeToRequest {
      _allowCache: true
    }

    interface MergeToResponse {
      page: number
      numPages: number
    }
  }
}
```
