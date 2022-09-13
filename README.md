# lambda-modules

Expose Node.js modules as Lambda functions

```
npm i lambda-modules
```

## Usage

Module function side:

```ts
// index.ts (lambda entrypoint)
import { createLambdaModuleHandler } from 'lambda-modules'

export const handler = createLambdaModuleHandler('./hello-service')
```

```ts
// hello-service.ts
export function helloWorld(input: string) {
  return `Hello World, ${input}`
}
```

Package the module:
```
$ npx package-lambda-module index.handler --package-name @org/hello-service --out-dir packages/hello-service/

Done! Lambda module path: packages/hello-service/package.json
```

Caller side: (needs IAM permission to call module function)

```
import * as helloService from '@org/hello-service'

async function main() {
  await helloService.helloWorld('Lambda') // returns "Hello World, Lambda"
}
```

