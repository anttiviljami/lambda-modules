# lambda-modules

Expose Node.js modules as Lambda functions

## Idea

Packages Node.js CommonJS or ESM modules into fully typed packages with module
exports converted into callable Lambda function invokes.

![Diagram](./lambda-modules.drawio.png)

## Installation

```
npm install lambda-modules
```

## Usage

Create a lambda handler for the module to export:

```ts
// index.ts (lambda entrypoint)
import { createLambdaModuleHandler } from 'lambda-modules'

export const handler = createLambdaModuleHandler('./hello-service')
```

This is the module we are exporting:

```ts
// hello-service.ts
export function helloWorld(input: string) {
  return `Hello World, ${input}`
}

export const HELLO_WORLD = 'values can be exported too!'
```

To package the module:

```
$ npx package-lambda-module index.handler --package-name @org/hello-service --out-dir packages/hello-service/
Done! Lambda module path: packages/hello-service/package.json
```

To use the module: (requires IAM permission to call lambda function)

```ts
import helloService from '@org/hello-service'

// configure module to call our lambda handler
helloService.__init({
  FunctionName: 'arn:aws:lambda:eu-central-1:123456789012:function:hello-service-module-handler',
})

async function main() {
  // all module exports are converted to async functions
  const result = await helloService.helloWorld('Lambda')
  console.log(result) // outputs "Hello World, Lambda"

  // exported values are also available as async functions
  const value = await helloService.HELLO_WORLD()
  console.log(value) // outputs "values can be exported too!"
}
```

## Implementation

The caller module uses [`Proxy`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
to intercept all calls to the module and handle them as lambda.invoke calls.

```ts
// pseudo-code
import { Lambda, ClientConfiguration } from 'aws-sdk'

class LambdaModule {
  public function __init(params: { FunctionName: string, lambdaConfig?: ClientConfiguration }) {
    this.lambda = new Lambda(params.lambdaConfig)
    this.FunctionName = params.FunctionName
  }
}

const lambdaModule = new LambdaModule()

const ModuleProxy = new Proxy(lambdaModule, {
  get(target, prop, receiver) {
    if (prop === '__init') return target[prop]

    return (...args) => target.lambda.invoke({
      FunctionName: target.FunctionName,
      Payload: {
        prop,
        args,
      },
    }).promise()
  }
})

export default ModuleProxy
```

The `createLambdaModuleHandler` outputs a handler that converts lambda invocations
to module calls.

```ts
// pseudo-code
interface LambdaModuleInvokeEvent {
  prop: string
  args?: any[]
}

export const createLambdaModuleHandler = (module: string) => async (event: LambdaModuleInvokeEvent) => {
  const { prop, args } = event

  const module = await import(module)

  if (typeof module[prop] === 'function') {
    return await module[prop](...args)
  }

  return module[prop]
}
```
