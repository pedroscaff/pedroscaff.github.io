---
title: "Nestjs Swagger Docs With Examples"
date: 2023-06-01T16:57:04+02:00
draft: true
---

NestJS has a great integration with OpenAPI/Swagger, leveraging decorators to add request and response information to endpoints. For more information on how to use it: https://docs.nestjs.com/openapi/introduction.

A simple response with example could look like this:

```typescript
import { ApiOperation, ApiProperty, ApiResponse  } from "@nestjs/swagger";
import {
  Controller,
  Get,
  HttpStatus,
} from "@nestjs/common";
import {} from "@nestjs/swagger";

class Cat {
  @ApiProperty({ example: "Ada" })
  name: string;
  @ApiProperty({ example: 10 })
  age: number;
}

const catStore = {
  "Ada": { name: "Ada", age: 10 }
};

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
  async getCatById(@Param("id") id: string) {
    const cat = catStore[id];
    if (!cat) {
      throw new NotFoundException("Cat not found");
    }
    return cat;
  }
}
```

The generated docs look like that:

![Generated docs with standard response](/static/response-standard.png)

## The Problem

I wanted to add a response with a custom type and specific examples for this endpoint, not the generic cat.
