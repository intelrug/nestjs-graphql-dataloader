# @intelrug/nestjs-graphql-dataloader

<a href="https://www.npmjs.com/package/@intelrug/nestjs-graphql-dataloader"><img src="https://img.shields.io/npm/v/@intelrug/nestjs-graphql-dataloader" alt="NPM Version" /></a>
<a href="https://www.npmjs.com/package/@intelrug/nestjs-graphql-dataloader"><img src="https://img.shields.io/npm/l/@intelrug/nestjs-graphql-dataloader" alt="Package License" /></a>
<a href="https://www.npmjs.com/package/@intelrug/nestjs-graphql-dataloader"><img src="https://img.shields.io/npm/dm/@intelrug/nestjs-graphql-dataloader" alt="NPM Downloads" /></a>

Based on https://github.com/krislefeber/nestjs-dataloader this small library assists in adding https://github.com/graphql/dataloader to a NestJS project.

This package also ensures that the ids are mapped to the dataloader in the correct sequence automatically and provides a helpful base class to simplify dataloader creation.

**Requires NestJS 7+**

### Install

```bash
$ yarn add @intelrug/nestjs-graphql-dataloader
```

### Usage

### 1. Register DataLoaderInterceptor
First, register a NestJS interceptor in your applications root module(s) providers configuration. This can actually be placed in any of your modules and it will be available anywhere but I would recommend your root module(s). It only needs to be defined once.

Add: 
```ts
{
  provide: APP_INTERCEPTOR,
  useClass: DataLoaderInterceptor,
}
```
    
For example:
```ts
import { DataLoaderInterceptor } from '@intelrug/nestjs-graphql-dataloader';
...

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: DataLoaderInterceptor,
    },
  ],
  
  ...
  imports: [
    RavenModule,
    ConfigModule.load(
      path.resolve(__dirname, '../../config', '**/!(*.d).{ts,js}'),
    ),
```

### 2. Build @Loaders for each @ObjectType

Using the provided template method, ```OrderedNestDataLoader<KeyType, EntityType>```, you can easily implement DataLoaders for your types. Here is an example:

```javascript
import { Injectable } from '@nestjs/common';
import { OrderedNestDataLoader } from '@intelrug/nestjs-graphql-dataloader';
import { Location } from '../core/location.entity';
import { LocationService } from '../core/location.service';

@Injectable()
export class LocationLoader extends OrderedNestDataLoader<Location['id'], Location> {
  constructor(private readonly locationService: LocationService) {
    super();
  }

  protected getOptions = () => ({
    query: (keys: Array<Location['id']>) => this.locationService.findByIds(keys),
  });
}

```
> *Note: In these examples the usage of ```Location['id']``` is referring to the type of the ```location.id property```, which in this case is ```string```. It would be perfectly acceptable to declare the generic type argument as ```string``` rather than ```Location['id']```.*

Add these to your modules providers as usual. You will most likely want to include it in your modules exports so the loader can be imported by resolvers in other modules.

```getOptions``` takes a single ```options``` argument which has the following interface:

```javascript
interface IOrderedNestDataLoaderOptions<ID, Type> {
  propertyKey?: string | string[];
  query: (keys: readonly ID[]) => Promise<Type[]>;
  typeName?: string;
}
```

Since the majority of the time a ```propertyKey``` is ```'id'``` this is the default if not specified. 

The ```typeName``` for the above example is automatically assigned ```'Location'``` which is derived from the class name, this is just used for logging errors.

The query is the equivalent of a ```repository.findByIds(ids)``` operation. It should return the **same number of elements** as requested. The **order does not matter** as the base loader implementation takes care of this.


### 3. Use the @Loader in @ResolveField

To then use the resolver it just needs to be injected into the resolvers field resolver method. Here is an example:

```ts
import DataLoader from 'dataloader';
...
@ResolveField(() => [Location])
public async locations(
  @Parent() company: Company,
  @Loader(LocationLoader)
  locationLoader: DataLoader<Location['id'], Location>,
) {
  return locationLoader.loadMany(company.locationIds);
}
```
