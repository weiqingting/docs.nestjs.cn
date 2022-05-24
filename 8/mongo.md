# Mongo

NEST支持与MongoDB数据库集成的两种方法。您可以使用此处描述的内置类型模块，该模块具有用于MongoDB的连接器，也可以使用Mongoose（最受欢迎的MongoDB对象建模工具）。在本章中，我们将使用专用的 @nestjs/mongoose软件包来描述后者。

首先下载所需的依赖

```js
$ npm install --save @nestjs/mongoose mongoose
```

安装完成之后，我们需要在 AppModule 中导出 MongooseModule 

```js
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

forRoot() 方法接受的参数，和 mongoose.connect() 一样

## 模型注入

在 mongoose 的世界里，所有的都是从 schema 推导出来，每个schema 都对应一个mongoDB 的集合或者对应 document 的结构。Schemas 用于定义模型，
模型用于创建和读取 MongoDB 数据库。

Schemas 是通过 NestJS 的修饰器创建的，或者通过使用 Mongoose 创建模式，使用装饰器制作可以大大降低样板并提高代码可读性

```js
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type CatDocument = Cat & Document;

@Schema()
export class Cat {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

注意，您还可以使用定义案例类别（来自Nestjs/Mongoose）生成原始模式定义。
这使您可以根据所提供的元数据手动修改生成的模式定义。这对于某些边缘案例很有用，在某些边缘箱中可能很难用装饰器代表所有内容。

@Schema() 装饰器将类标记为模式定义，他将 Cat Class 映射同名的 MongoDB 集合中，但一般对应的集合名要带上s，最终的Mongo Collection名称将是Cats。
该装饰器接受单个可选参数，该参数是架构选项对象。new mongoose.Schema(_, options)

@prop（）装饰器定义文档中的属性。例如，在上面的模式定义中，我们定义了三个属性：name，age 和breed。
由于Typescript元数据（和反射）功能，这些属性的架构类型将自动推断。但是，在更复杂的方案中，不能隐式反映类型（例如，数组或嵌套对象结构），必须明确指示类型，如下：

```js
@Prop([String])
tags: string[];
```

@props() 装饰器接受一个参数，比如可以定义字段是否必填，默认值，或者一些特殊处理等

```js
@Prop({ required: true })
name: string;
```

你可以通过Prop（），想把某个属性关联另一个模型。

```js
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

// inside the class definition
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;

```
如果有多个owners

```js
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owner: Owner[];
```
最后，可以将原始的定义传给装饰器，这个非常有用，比如，一个模型并没有在nest中定义模型，你就可以用 row 临时定义

```js
@Prop(raw({
  firstName: { type: String },
  lastName: { type: String }
}))
details: Record<string, any>;
```

如果你没有用装饰器，你可以手动定义schema 
```js
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```
我们建议把 cat.schema 放在 CatsModule 同一个目录中

```js
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { Cat, CatSchema } from './schemas/cat.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }])],
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

MongooseModule 提供了 forFeature（） 方法，主要用于注册那些定义模型，
如果你想在另一个模块中使用这些模型，请将 MongooseModule 添加到 CatsModule 的导出部分，并在另一个模块中使用他

只要注册一次，你可以通过 @InjectModel（） 引入 Cat，从而在 CatService 中使用

```js
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Cat, CatDocument } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name) private catModel: Model<CatDocument>) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
```

### Connection

有时，你可能需要访问本机 Mongoose Connection 对象，例如，你可能需要在链接对象上进行本机API调试，你可以使用 @InjectConnection()
装饰器注入 Mongoose 链接

```js
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}

```

### Multiple databases

一些项目需要多个数据库连接。该模块也可以实现这一点。要使用多个连接，请首先创建连接。在这种情况下，连接命名将成为强制性。

```js

import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionName: 'cats',
    }),
    MongooseModule.forRoot('mongodb://localhost/users', {
      connectionName: 'users',
    }),
  ],
})
export class AppModule {}
```
> 请注意，如果没有名称或相同名称，您不应具有多个连接，否则它们将被覆盖。

使用此设置后，你必须告诉 MongooseModule.forFeature() 使用哪个链接

```js
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class AppModule {}
```
你也可以注入给定的链接

```js
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection('cats') private connection: Connection) {}
}
```
对于自定义提供者，可以通过 getConnectionToken（） 传递链接

```js
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```
### Hooks (middleware)

中间件是通过useFactory 的一个异步函数，用于一些能力的强化。你可以调用 pre() 或者 post() 进行注册钩子

```js
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.pre('save', function () {
            console.log('Hello from pre save');
          });
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```


