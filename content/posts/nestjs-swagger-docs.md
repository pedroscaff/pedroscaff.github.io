---
title: "Nestjs Swagger Docs With Examples"
date: 2023-06-01T16:57:04+02:00
draft: false
---

NestJS has a great integration with OpenAPI/Swagger, leveraging decorators to add request and response information to endpoints. For more information on how to use it: https://docs.nestjs.com/openapi/introduction.

A simple response with example could look like this:

```typescript
import { ApiOperation, ApiProperty, ApiResponse  } from "@nestjs/swagger";
import {
  Controller,
  Get,
  HttpStatus,
  NotFoundException,
  Param,
} from "@nestjs/common";

class Cat {
  @ApiProperty({ example: "Ada" })
  name: string;
  @ApiProperty({ example: 10 })
  age: number;
  @ApiProperty({ example: true })
  adoptable: boolean;
  @ApiProperty({ example: false })
  isAdopted: boolean;
}

const catStore: Map<string,Cat> = new Map<string,Cat>();
catStore["Ada"] = { name: "Ada", age: 10, adoptable: true, isAdopted: false }

@Controller({
  path: "/cat",
})
export class CatController {
  @Get("/:id")
  @ApiOperation({ summary: "My example get cat by id endpoint" })
  @ApiResponse({ status: HttpStatus.OK, type: Cat })
  @ApiResponse({
    status: HttpStatus.NOT_FOUND,
    description: "Cat not found",
  })
  getCatById(@Param("id") id: string): Cat {
    const cat = catStore.get(id);
    if (!cat) {
      throw new NotFoundException("Cat not found");
    }
    return cat;
  }
}
```

The generated docs look like that:

![Generated docs with standard response](/images/response-standard.png)

## The Problem

Let's say we want to have an endpoint to adopt a cat. Adopting a cat might have constrains we need to take into account and inform the user about.

For this example we will add two constrains: the cat hasn't been adopted yet, and the cat wants to be adopted. We can define the "violations" for trying to adopt a cat that cannot be adopted and return them:

```typescript
export class AdoptCatViolation {
  @ApiProperty({ example: "CAT_ALREADY_ADOPTED" })
  type: string;
  @ApiProperty({ example: "Cat already adopted" })
  description: string;
}

...

@Post("/:id/adopt")
@ApiOperation({ summary: "Adopt cat by id endpoint" })
@ApiResponse({ status: HttpStatus.OK, type: Cat })
@ApiResponse({ status: HttpStatus.UNPROCESSABLE_ENTITY, type: AdoptCatViolation })
@ApiResponse({
  status: HttpStatus.NOT_FOUND,
  description: "Cat not found",
})
adoptCatById(@Param("id") id: string): Cat {
  const cat = catStore.get(id);
  if (!cat) {
    throw new NotFoundException("Cat not found");
  }
  if (!cat.adoptable) {
    throw new UnprocessableEntityException(
      { type: "CAT_WANTS_TO_STAY_FREE", description: "Cat wants to stay free" }
    )
  }
  if (cat.isAdopted) {
    throw new UnprocessableEntityException(
      { type: "CAT_ALREADY_ADOPTED", description: "Cat already adopted" }
    )
  }
  cat.isAdopted = true;
  return cat;
}
```

We have handled the errors that can happen and returned them to the user. The documentation, however, only shows the generic `AdoptCatViolation` example. It is also possible to set an `examples` object (`@ApiResponse({ examples: { ... } })`) with more than one value, but the generated examples in the documentation don't show them. This is what the Swagger docs look like:

![Generated docs with default example](/images/response-default-example.png)

This can be enough for some cases, but I want to be able to generate the docs with all violations I know I might return, allowing the user to know about them before hand and handle them properly.

## The solution

We have to go deeper into the params of `@ApiResponse({ ... })` to specify the schema and the examples in the content field of the `ApiResponseOptions`. I wrapped it in a function to make it reusable:

```typescript
import { ApiResponseOptions, getSchemaPath } from "@nestjs/swagger";
import { ExamplesObject } from '@nestjs/swagger/dist/interfaces/open-api-spec.interface';

private static createViolationResponseDocs(description: string, examples: ExamplesObject): ApiResponseOptions {
  return {
    status: HttpStatus.UNPROCESSABLE_ENTITY,
    description,
    content: {
      "application/json": {
        schema: { $ref: getSchemaPath(AdoptCatViolation) },
        examples,
      }
    }
  }
}
```

Let's create a new adopt cat endpoint using the function. First we define the violations:

```typescript
const adoptCatViolations: ExamplesObject = {
  "CAT_ALREADY_ADOPTED": {
    value: {
      type: "CAT_ALREADY_ADOPTED",
      description: "Cat already adopted",
    } as AdoptCatViolation
  },
  "CAT_WANTS_TO_STAY_FREE": {
    value: {
      type: "CAT_WANTS_TO_STAY_FREE",
      description: "Cat wants to stay free",
    } as AdoptCatViolation
  },
};
```

Now we can use the function to define the docs, and also use the violations as the param for the `UnprocessableEntityException` constructor.

```typescript
@Post("/:id/adopt-with-nice-docs")
@ApiOperation({ summary: "Adopt cat by id endpoint" })
@ApiResponse({ status: HttpStatus.OK, type: Cat })
@ApiResponse(CatController.createViolationResponseDocs(
  "Cat already adopted or wants to stay free!",
  adoptCatViolations,
))
@ApiResponse({
  status: HttpStatus.NOT_FOUND,
  description: "Cat not found",
})
adoptCatByIdWithNiceDocs(@Param("id") id: string): Cat {
  const cat = catStore.get(id);
  if (!cat) {
    throw new NotFoundException("Cat not found");
  }
  if (!cat.adoptable) {
    throw new UnprocessableEntityException(
      adoptCatViolations["CAT_WANTS_TO_STAY_FREE"]
    )
  }
  if (cat.isAdopted) {
    throw new UnprocessableEntityException(
      adoptCatViolations["CAT_ALREADY_ADOPTED"]
    )
  }
  cat.isAdopted = true;
  return cat;
}
```

Now we have the docs with the possible violations as a dropdown menu for this endpoint:

![Generated docs with custom response](/images/response-custom.png)

### Schema path

If the schema isn't directly referenced as a response type (e.g. `@ApiResponse({ type: AdoptCatViolation })`), it has to be added to the NestJS app like that:

```typescript
...
const options: SwaggerDocumentOptions = {
  extraModels: [AdoptCatViolation],
};
const document = SwaggerModule.createDocument(app, config, options);
...
```

For more information on it, see https://docs.nestjs.com/openapi/types-and-parameters#extra-models.

## Final words

I hope this example helps others, it is a simple task but that took me a bit of research to find out how to do it. Here is a link to the complete example: https://git.sr.ht/~pedroscaff/nestjs-swagger-docs-example
