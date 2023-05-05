<p align="center">
  <a href="http://nestjs.com/" target="blank"><img src="https://nestjs.com/img/logo_text.svg" width="320" alt="Nest Logo" /></a>
</p>

[travis-image]: https://api.travis-ci.org/nestjs/nest.svg?branch=master
[travis-url]: https://travis-ci.org/nestjs/nest
[linux-image]: https://img.shields.io/travis/nestjs/nest/master.svg?label=linux
[linux-url]: https://travis-ci.org/nestjs/nest

  <p align="center">A progressive <a href="http://nodejs.org" target="blank">Node.js</a> framework for building efficient and scalable server-side applications.</p>
    <p align="center">
<a href="https://www.npmjs.com/~nestjscore"><img src="https://img.shields.io/npm/v/@nestjs/core.svg" alt="NPM Version" /></a>
<a href="https://www.npmjs.com/~nestjscore"><img src="https://img.shields.io/npm/l/@nestjs/core.svg" alt="Package License" /></a>
<a href="https://www.npmjs.com/~nestjscore"><img src="https://img.shields.io/npm/dm/@nestjs/core.svg" alt="NPM Downloads" /></a>
<a href="https://travis-ci.org/nestjs/nest"><img src="https://api.travis-ci.org/nestjs/nest.svg?branch=master" alt="Travis" /></a>
<a href="https://travis-ci.org/nestjs/nest"><img src="https://img.shields.io/travis/nestjs/nest/master.svg?label=linux" alt="Linux" /></a>
<a href="https://coveralls.io/github/nestjs/nest?branch=master"><img src="https://coveralls.io/repos/github/nestjs/nest/badge.svg?branch=master#5" alt="Coverage" /></a>
<a href="https://discord.gg/G7Qnnhy" target="_blank"><img src="https://img.shields.io/badge/discord-online-brightgreen.svg" alt="Discord"/></a>
<a href="https://opencollective.com/nest#backer"><img src="https://opencollective.com/nest/backers/badge.svg" alt="Backers on Open Collective" /></a>
<a href="https://opencollective.com/nest#sponsor"><img src="https://opencollective.com/nest/sponsors/badge.svg" alt="Sponsors on Open Collective" /></a>
  <a href="https://paypal.me/kamilmysliwiec"><img src="https://img.shields.io/badge/Donate-PayPal-dc3d53.svg"/></a>
  <a href="https://twitter.com/nestframework"><img src="https://img.shields.io/twitter/follow/nestframework.svg?style=social&label=Follow"></a>
</p>
  <!--[![Backers on Open Collective](https://opencollective.com/nest/backers/badge.svg)](https://opencollective.com/nest#backer)
  [![Sponsors on Open Collective](https://opencollective.com/nest/sponsors/badge.svg)](https://opencollective.com/nest#sponsor)-->

## Description
---
This is forked from https://github.com/nestjs/azure-database and modified to work only with Azure Cosmos DB.
**The credit goes to the original author.**

### For Azure Cosmos DB support

1. Create or update your existing `.env` file with the following content:

```
AZURE_COSMOS_DB_NAME=
AZURE_COSMOS_DB_ENDPOINT=
AZURE_COSMOS_DB_KEY=
```

2. **IMPORTANT: Make sure to add your `.env` file to your .gitignore! The `.env` file MUST NOT be versioned on Git.**

3. Make sure to include the following call to your main file:

```typescript
if (process.env.NODE_ENV !== 'production') require('dotenv').config();
```

> This line must be added before any other imports!

### Example

> Note: Check out the CosmosDB example project included in the [sample folder](https://github.com/nestjs/azure-database/tree/master/sample/cosmos-db)

#### Prepare your entity

0. Create a new feature module, eg. with the nest CLI:

```shell
$ nest generate module event
```

1. Create a Data Transfer Object (DTO) inside a file named `event.dto.ts`:

```typescript
export class EventDTO {
  name: string;
  type: string;
  date: Date;
  location: Point;
}
```

2. Create a file called `event.entity.ts` and describe the entity model using the provided decorators:

- `@CosmosPartitionKey(value: string)`: Represents the `PartitionKey` of the entity (**required**).

- `@CosmosDateTime(value?: string)`: For DateTime values.

For instance, the shape of the following entity:

```typescript
import { CosmosPartitionKey, CosmosDateTime, Point } from '@nestjs/azure-database';

@CosmosPartitionKey('type')
export class Event {
  id?: string;
  type: string;
  @CosmosDateTime() createdAt: Date;
  location: Point;
}
```

Will be automatically converted to:

```json
{
  "type": "Meetup",
  "createdAt": "2019-11-15T17:05:25.427Z",
  "position": {
    "type": "Point",
    "coordinates": [2.3522, 48.8566]
  }
}
```

3. Import the `AzureCosmosDbModule` inside your Nest feature module `event.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { AzureCosmosDbModule } from '@nestjs/azure-database';
import { EventController } from './event.controller';
import { EventService } from './event.service';
import { Event } from './event.entity';

@Module({
  imports: [
    AzureCosmosDbModule.forRoot({
      dbName: process.env.AZURE_COSMOS_DB_NAME,
      endpoint: process.env.AZURE_COSMOS_DB_ENDPOINT,
      key: process.env.AZURE_COSMOS_DB_KEY,
    }),
    AzureCosmosDbModule.forFeature([{ dto: Event }]),
  ],
  providers: [EventService],
  controllers: [EventController],
})
export class EventModule {}
```

#### CRUD operations

0. Create a service that will abstract the CRUD operations:

```shell
$ nest generate service event
```

1. Use the `@InjectModel(Event)` to get an instance of the Azure Cosmos DB [Container](https://docs.microsoft.com/en-us/javascript/api/@azure/cosmos/container) for the entity definition created earlier:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/azure-database';
import { Event } from './event.entity';

@Injectable()
export class EventService {
  constructor(
    @InjectModel(Event)
    private readonly eventContainer,
  ) {}
}
```

The Azure Cosmos DB `Container` provides a couple of public APIs and Interfaces for managing various CRUD operations:

##### CREATE

`create(entity: T): Promise<T>`: creates a new entity.

```typescript

  @Post()
  async create(event: Event): Promise<Event> {
      return this.eventContainer.items.create(event)
  }

```

##### READ

`query<T>(query: string | SqlQuerySpec, options?: FeedOptions): QueryIterator<T>`: run a SQL Query to find a document.

```typescript
  @Get(':id')
  async getContact(@Param('id') id) {
    try {
       const querySpec = {
           query: "SELECT * FROM root r WHERE r.id=@id",
           parameters: [
             {
               name: "@id",
               value: id
             }
           ]
         };
        const { resources } = await this.eventContainer.items.query<Event>(querySpec).fetchAll()
         return resources
    } catch (error) {
      // Entity not found
      throw new UnprocessableEntityException(error);
    }
  }
```

##### UPDATE

`read<T>(options?: RequestOptions): Promise<ItemResponse<T>>`: Get a document.
`replace<T>(body: T, options?: RequestOptions): Promise<ItemResponse<T>>`: Updates a document.

```typescript
  @Put(':id')
  async saveEvent(@Param('id') id, @Body() eventData: EventDTO) {
    try {
      const { resource: item } = await this.eventContainer.item<Event>(id, 'type').read()

      // Disclaimer: Assign only the properties you are expecting!
      Object.assign(item, eventData);

      const { resource: replaced } = await this.eventContainer
       .item(id, 'type')
       .replace<Event>(item)
      return replaced
    } catch (error) {
      throw new UnprocessableEntityException(error);
    }
  }
```

##### DELETE

`delete<T>(options?: RequestOptions): Promise<ItemResponse<T>>`: Removes an entity from the database.

```typescript

  @Delete(':id')
  async deleteEvent(@Param('id') id) {
    try {
      const { resource: deleted } = await this.eventContainer
       .item(id, 'type')
       .delete<Event>()

      return deleted;
    } catch (error) {
      throw new UnprocessableEntityException(error);
    }
  }
```

## Support

Nest is an MIT-licensed open source project. 

## License

Nest is [MIT licensed](LICENSE).

[edm-types]: http://bit.ly/nest-edm
