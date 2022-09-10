# topic-prisma-aws-lambda-deployment

This repo is an example of how to deploying AWS lambdas with Lambda layers ft. TypeScript, Prisma and Serverless.

Blog post: https://dev.to/eddeee888/how-to-deploy-prisma-in-aws-lambda-with-serverless-1m76

## Requirements

- [AWS account](https://aws.amazon.com/account/) - free-tier
- [Serverless account](https://www.serverless.com/) - free-tier
- One AWS RDS Database \*
- One IAM user with access key ID and secret access key \*\*

\* A publicly accessible database was used to test the Lambda in this guide. Make sure you have appropriate permissions on your own database.

\*\* An IAM user with full admin access was used to test the Lambda in this guide. Make sure you use appropriate permissions for your IAM user

## How it works.

### AWS lambda and lambda layers

An AWS Lambda has a size limit of 50MB. It may seem like a lot but if we build a Lambda with all the dependencies in `node_modules` ( which may include the generated Prisma Client ), it would easily go over the limit.

A good practice is to keep only the main business logic in the Lambda function. Keeping a Lambda small means it takes less time to deploy. All imports should be treated as **external** i.e. as if they come from `node_modules`.

All imports can come from Lambda layers ( up to 5 layers ). In this repo, there are 3 types of layers:

- Runtime dependencies - The only dependency runtime dependency here is `uuid` for testing purpose. It should be the only one in `package.json`'s dependencies field.
- `@prisma/*` - I prefer keeping `@prisma/*` and `.prisma` as its own layer because the way it's created is fairly different from other dependencies.
- `@libs/*` - This includes utility functions that can be shared between lambdas and other apps.

### Prisma binary and AWS Lambda

Prisma can generate different binaries for different runtime environment. This can be changed using the `binaryTargets` field in the prisma schema. For example:

```
generator client {
  provider      = "prisma-client-js"
  binaryTargets = ["darwin"] // For MacOS
}
```

A client binary only works if used in the environment it's intended for. For example, a client generated on Windows will not work in AWS Lambda. If we declare multiple `binaryTargets`, Prisma will generate multiple clients, increasing the total package size.

We want to use `native` when working locally and `rhel-openssl-1.0.x` for Lambdas. One way to approach this is to use an environment variable:

```
generator client {
  provider      = "prisma-client-js"
  binaryTargets = [env("PRISMA_BINARY_TARGET")]
}
```

### Serverless and AWS lambda

[Serverless](https://www.serverless.com/) is a powerful framework that can help deploy lambdas and apps easily. We will use this service to orchestrate the deployment of our lambda and lambda layers using a [serverless.yml file](./serverless.yml)

## Lambda functions

Our lambda functions are located in [src/lambdas](./src/lambdas). This will be built into `build/lambdas/*` using TypeScript. Check out this [example lambda that inserts a new user](./src/lambdas/insertUser/handler.ts)

### Using @libs/\*

In the sample Lambda, we are importing a function to create Prisma Client:

```ts
import { createPrismaClient } from "@libs/prismaClient";
```

Locally, we use TypeScript to map everything starting with `@libs/*` to [src/libs](./src/libs) using [tsconfig paths config](https://github.com/eddeee888/topic-prisma-aws-lambda-deployment/blob/a80ad9ba5131b31ee321a23777a2c5f83332059d/tsconfig.json#L6-L8)

The prod Lambda function will import `@libs/*` from a Lambda layer ( more on this later ).

## Creating Lambda layers

Each layer should be zipped up before sending to AWS. Layers of a Node.js should have the following structure:

```
layer.zip
  -- nodejs
       -- node_modules
            -- lib1
            -- lib2
  ...
```

You can read more on lambda layers [here](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html)

In our case, we will split our Lambda layers like this:

```
lambda-layers-node_modules.zip
  -- nodejs
       -- node_modules
            -- uuid
```

```
lambda-layers-prisma-client.zip
  -- nodejs
       -- node_modules
            -- .prisma
            -- @prisma
```

```
lambda-layers-libs.zip
  -- nodejs
       -- node_modules
            -- @libs
```

Check out the following scripts that are intended to be run in CI to create the mentioned zip files:

- [prepare-node-modules-lambda-layer.sh](./scripts/ci/prepare-node-modules-lambda-layer.sh)

- [prepare-prisma-client-lambda-layer.sh](./scripts/ci/prepare-prisma-client-lambda-layer.sh)

- [prepare-libs-lambda-layer.sh](./scripts/ci/prepare-libs-lambda-layer.sh)

## Putting it all together and deploy

You can use this sample [github action](./.github/workflows/deploy-lambdas.yml) to deploy the Lambda functions, together with their layers. Here's a summary of what it does:

- Build `node_modules` Kambda layer
- Build `@prisma/*` Lambda layer
- Build `@libs/*` Lambda layer
- Build lambda functions
- Once all the previous steps are done, download all built assets and deploy using [Serverless](./serverless.yml)

**Note**

- Each Lambda layer build step has been customised to avoid overlapping of dependencies in each layer.
- Each Lambda layer [is unzipped before being deployed](https://github.com/eddeee888/topic-prisma-aws-lambda-deployment/blob/1738d2ae2e1a6a44b45eefb76bc19d02254b4c41/.github/workflows/deploy-lambdas.yml#L151-L158) because Serverless zips them by default.
- All assets and `serverless.yml` are moved into `./build/lambdas`. This is because the [SERVICE_ROOT](https://github.com/eddeee888/topic-prisma-aws-lambda-deployment/blob/a80ad9ba5131b31ee321a23777a2c5f83332059d/.github/workflows/deploy-lambdas.yml#L168) option seems to tell serverless include more than it should if `./` is used
- When deploying, there are some [AWS and Serverless account env variables](https://github.com/eddeee888/topic-prisma-aws-lambda-deployment/blob/a80ad9ba5131b31ee321a23777a2c5f83332059d/.github/workflows/deploy-lambdas.yml#L169-L171) and some are used to map [environment variables for the lambda functions](https://github.com/eddeee888/topic-prisma-aws-lambda-deployment/blob/a80ad9ba5131b31ee321a23777a2c5f83332059d/.github/workflows/deploy-lambdas.yml#L172-L174) in [serverless.yml](https://github.com/eddeee888/topic-prisma-aws-lambda-deployment/blob/a80ad9ba5131b31ee321a23777a2c5f83332059d/serverless.yml#L32-L33)

---

Made with ❤️ by Eddy Nguyen
https://eddeee888.me

This repo is extracted from https://github.com/eddeee888/base-app-monorepo
