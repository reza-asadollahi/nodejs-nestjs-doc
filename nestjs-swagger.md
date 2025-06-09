# Set up swagger on the NestJs project:

First install ``@nestjs/swagger`` library, then add following code to the ``main.ts``:
``` typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.setGlobalPrefix('api');

  // Swagger setup
  const config = new DocumentBuilder()
    .setTitle('Element Metadata Service')
    .setBasePath('api')
    .setDescription('API for syncing and querying element metadata')
    .setVersion('1.0.0')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document);

  await app.listen(process.env.PORT ?? 3000);
  console.log(`Application is running on: ${await app.getUrl()}`);
}

bootstrap();
```



# Config each API
In NestJS, Swagger documentation is provided using decorators from the @nestjs/swagger package. Here are examples of different types of documentation you can add:


## 1. Basic API Operation Documentation (@ApiOperation)
Provides a summary of what the endpoint does:
``` typescript
@ApiOperation({ summary: 'Retrieve a brief element by tag or ID' })
@Get('brief')
async getBriefElement(
  @Query('tagName') tagName?: string, 
  @Query('elementId') elementId?: string
) {
  const result = await this.elementsService.getBriefElement({ tagName, elementId });
  if (!result) throw new HttpException('Element not found', 404);
  return result;
}
```

## 2. Tagging Controllers for Organization (@ApiTags)
Organizes APIs in Swagger under specific categories:
``` typescript
@ApiTags('Brief Elements')
@Controller('elements')
export class ElementsController {
  // API methods go here
}
```

## 3. Parameter Documentation (@ApiQuery & @ApiParam)
Documents query parameters to clarify usage:
``` typescript
@ApiQuery({ name: 'tagName', required: false, type: String, description: 'Search by tag name' })
@ApiQuery({ name: 'elementId', required: false, type: String, description: 'Search by element ID' })
@Get('brief')
async getBriefElement(
  @Query('tagName') tagName?: string, 
  @Query('elementId') elementId?: string
) {
  const result = await this.elementsService.getBriefElement({ tagName, elementId });
  if (!result) throw new HttpException('Element not found', 404);
  return result;
}
```


## 4. Response Type Documentation (@ApiResponse)
Specifies response codes and descriptions:
``` typescript
@ApiResponse({ status: 200, description: 'Successfully retrieved brief element' })
@ApiResponse({ status: 404, description: 'Element not found' })
@Get('brief')
async getBriefElement(
  @Query('tagName') tagName?: string, 
  @Query('elementId') elementId?: string
) {
  const result = await this.elementsService.getBriefElement({ tagName, elementId });
  if (!result) throw new HttpException('Element not found', 404);
  return result;
}
```


## 5. DTO Documentation (@ApiProperty)
Documents DTOs for request/response models:
``` typescript
import { ApiProperty } from '@nestjs/swagger';

export class BriefElementDto {
  @ApiProperty({ example: 'header', description: 'Tag name of the element' })
  tagName: string;

  @ApiProperty({ example: '12345', description: 'Unique element ID' })
  elementId: string;
}
```


## 6. Security Documentation (@ApiBearerAuth)
Marks APIs that require authentication:
``` typescript
@ApiBearerAuth()
@ApiOperation({ summary: 'Protected API requiring authentication' })
@Get('protected')
async protectedEndpoint() {
  return { message: 'You accessed a protected endpoint!' };
}
```
