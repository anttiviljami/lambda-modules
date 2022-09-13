# lambda-modules

Expose Node.js modules as Lambda functions

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

To use the module: (needs IAM permission to call lambda function)

```ts
import * as helloService from '@org/hello-service'

// configure lambda module
helloService.__init({
  FunctionName: 'arn:aws:lambda:eu-central-1:123456789012:function:hello-service-handler',
})

async function main() {
  const result = await helloService.helloWorld('Lambda') // all module exports are converted to async functions
  console.log(result) // outputs "Hello World, Lambda"

  const value = await helloService.HELLO_WORLD() // exported values are also available as async functions
  console.log(value) // outputs "values can be exported too!"
}
```

