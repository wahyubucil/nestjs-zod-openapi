# NestJS Zod OpenAPI

> NestJS helper to easily use Zod with OpenAPI

[![npm-image]][npm-url] [![license-image]][license-url] [![typescript-image]][typescript-url]

This package is a combination of two awesome packages:

- [@anatine/zod-nestjs](https://www.npmjs.com/package/@anatine/zod-nestjs): provide a validation pipe on data and helper methods to create DTO from a Zod schema.
- [@asteasolutions/zod-to-openapi](https://www.npmjs.com/package/@asteasolutions/zod-to-openapi): provide `openapi` method to the Zod schema. The advantage of this package is the built-in schema reference, so when we have a nested schema, the OpenAPI documentation will reference the schema instead of just creating a plain new object.

Most of the documentation here will be adopted from both of the packages. If something's missing here, please refer to each package's documentation. Thanks to both of the authors for making such awesome packages.

## Table of contents

- [Installation](#installation)
- [Usage](#usage)
  - [Set up your app](#set-up-your-app)
  - [Using `ZodValidationPipe`](#using-zodvalidationpipe)
  - [Generate a schema](#generate-a-schema)
  - [Use the schema in controller](#use-the-schema-in-controller)

## Installation

Make sure you've set up the `@nestjs/swagger` module first. [Read it here](https://docs.nestjs.com/openapi/introduction).

After set up is completed you can install this package and the [Zod][zod-url] package:

```sh
# NPM
npm i zod @wahyubucil/nestjs-zod-openapi
# Yarn
yarn add zod @wahyubucil/nestjs-zod-openapi
# PNPM
pnpm add zod @wahyubucil/nestjs-zod-openapi
```

Zod requires us to enable `strict` mode in your `tsconfig.json`.

```json
// tsconfig.json
{
  // ...
  "compilerOptions": {
    // ...
    "strict": true
  }
}
```

Example `tsconfig.json`:

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strict": true
  }
}
```

## Usage

### Set up your app

1. Make sure you load the `@wahyubucil/nestjs-zod-openapi/boot` script from this package at the top of the main script.
2. Patch the swagger so that it can use Zod types before you create the document.

Example Main App:

```ts
import '@wahyubucil/nestjs-zod-openapi/boot' // <-- add this. The boot script should be on the top of this file.

import { NestFactory } from '@nestjs/core'
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger'
import { patchNestjsSwagger } from '@wahyubucil/nestjs-zod-openapi' // <-- add this. Import the patch for NestJS Swagger

import { AppModule } from './app.module'

async function bootstrap() {
  const app = await NestFactory.create(AppModule)

  const config = new DocumentBuilder()
    .setTitle('Example API')
    .setDescription('The example API description')
    .setVersion('1.0')
    .build()

  patchNestjsSwagger({ schemasSort: 'alpha' }) // <-- add this. This function should run before the `SwaggerModule.createDocument` function.

  const document = SwaggerModule.createDocument(app, config)
  SwaggerModule.setup('docs', app, document)

  await app.listen(3000)
}
bootstrap()
```

The `patchNestjsSwagger` contains an options:

- `schemasSort`: to determine the sorting mechanism
  - `default`: no special sorting mechanism, it's based on the schema declaration order.
  - `alpha`: sorting based on the alpha-numeric order.
  - `localeCompare`: using JavaScript `localeCompare` to sort the order, it will use each locale sorting mechanism.

### Using `ZodValidationPipe`

There are two ways to do it:

- Globally (recommended):

```ts
// app.module.ts
import { APP_PIPE } from '@nestjs/core'
import { ZodValidationPipe } from '@wahyubucil/nestjs-zod-openapi'

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ZodValidationPipe,
    },
  ],
})
export class AppModule {}
```

- Locally

```ts
// cats.controller.ts
import { ZodValidationPipe } from '@wahyubucil/nestjs-zod-openapi'

// controller-level
@UsePipes(ZodValidationPipe)
class CatsController {}

class CatsController {
  // route-level
  @UsePipes(ZodValidationPipe)
  async create() {}
}
```

### Generate a schema

Use [Zod][zod-url] to generate a schema.

Example schema:

```ts
// cats.dto.ts
import { createZodDto } from '@wahyubucil/nestjs-zod-openapi'
import { z } from 'zod'

// This will make a new Zod schema with the name 'User'
export const User = z
  .object({
    email: z.string(),
    displayName: z
      .string()
      .openapi({ description: 'Display name of the user' }), // <-- using the `openapi` method to add additional info on the field
  })
  .openapi('User')

// This will make a new Zod schema with the name 'CatShelter'
export const Shelter = z
  .object({
    name: z.string(),
    address: z.string(),
  })
  .openapi('CatShelter', { description: 'Shelter information' }) // <-- using the `openapi` method to add additional info on the schema

// Example DTO Schemas to be used on controller
export const Cat = z.object({
  name: z.string(),
  age: z.number(),
  breed: z.string(),
  shelter: Shelter,
  createdBy: User,
})
export class CatDto extends createZodDto(Cat) {}

export class UpdateCatDto extends createZodDto(
  Cat.omit({ name: true, createdBy: true }),
) {}

export const GetCats = z.object({
  cats: z.array(z.string()),
})
export class GetCatsDto extends createZodDto(GetCats) {}

export const CreateCatResponse = z.object({
  success: z.boolean(),
  message: z.string(),
  name: z.string(),
})
export class CreateCatResponseDto extends createZodDto(CreateCatResponse) {}

export class UpdateCatResponseDto extends createZodDto(
  CreateCatResponse.omit({ name: true }),
) {}
```

### Use the schema in controller

This follows the standard NestJS method of creating controllers.

`@nestjs/swagger` decorators should work normally.

Example Controller:

```ts
// cats.controller.ts
import { Body, Controller, Get, Param, Patch, Post } from '@nestjs/common'
import { ApiCreatedResponse, ApiOkResponse } from '@nestjs/swagger'
import {
  CatDto,
  CreateCatResponseDto,
  GetCatsDto,
  UpdateCatDto,
  UpdateCatResponseDto,
} from './cats.dto'

@Controller('cats')
export class CatsController {
  // Use DTO as the type-safety helper
  @Get()
  @ApiOkResponse({
    type: GetCatsDto,
  })
  async findAll(): Promise<GetCatsDto> {
    return { cats: ['Lizzie', 'Spike'] }
  }

  // If you want both runtime and type safety, we recommend to parse the schema directly instead of using it as a return type
  @Get(':id')
  @ApiOkResponse({
    type: CatDto,
  })
  async findOne(@Param() { id }: { id: string }) {
    return CatDto.zodSchema.parse({
      name: `Cat-${id}`,
      age: 8,
      breed: 'Unknown',
      shelter: {
        name: 'A sample shelter',
        address: 'Bali, Indonesia',
      },
      createdBy: {
        email: 'email@example.com',
        displayName: 'Wahyu Bucil',
      },
    })
  }

  @Post()
  @ApiCreatedResponse({
    description: 'The record has been successfully created.',
    type: CreateCatResponseDto,
  })
  async create(@Body() createCatDto: CatDto) {
    return CreateCatResponseDto.zodSchema.parse({
      success: true,
      message: 'Cat created',
      name: createCatDto.name,
    })
  }

  @Patch()
  @ApiOkResponse({
    type: UpdateCatDto,
  })
  async update(@Body() updateCatDto: UpdateCatDto) {
    return UpdateCatResponseDto.zodSchema.parse({
      success: true,
      message: `Cat's age of ${updateCatDto.age} updated`,
    })
  }
}
```

NOTE: You still need to explicitly define the response for the OpenAPI via the decorators provided by `@nestjs/swagger`.

---

<div align="center">
  <sub>Built with ❤︎ by <a href="https://twitter.com/wahyubucil">Wahyu "The GOAT" Bucil</a>
</div>

[npm-image]: https://img.shields.io/npm/v/@wahyubucil/nestjs-zod-openapi.svg?style=for-the-badge&logo=npm
[npm-url]: https://npmjs.org/package/@wahyubucil/nestjs-zod-openapi 'npm'
[license-image]: https://img.shields.io/npm/l/@wahyubucil/nestjs-zod-openapi?color=blueviolet&style=for-the-badge
[license-url]: LICENSE.md 'license'
[typescript-image]: https://img.shields.io/badge/Typescript-294E80.svg?style=for-the-badge&logo=typescript
[typescript-url]: "typescript"
[zod-url]: https://github.com/colinhacks/zod
