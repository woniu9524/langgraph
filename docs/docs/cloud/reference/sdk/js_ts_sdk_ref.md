<a name="readmemd"></a>

**[@langchain/langgraph-sdk](https://github.com/langchain-ai/langgraph/tree/main/libs/sdk-js)**

***

## [@langchain/langgraph-sdk](https://github.com/langchain-ai/langgraph/tree/main/libs/sdk-js)

### Classes

- [AssistantsClient](#classesassistantsclientmd)
- [Client](#classesclientmd)
- [CronsClient](#classescronsclientmd)
- [RunsClient](#classesrunsclientmd)
- [StoreClient](#classesstoreclientmd)
- [ThreadsClient](#classesthreadsclientmd)

### Interfaces

- [ClientConfig](#interfacesclientconfigmd)

### Functions

- [getApiKey](#functionsgetapikeymd)


<a name="authreadmemd"></a>

**@langchain/langgraph-sdk**

***

## @langchain/langgraph-sdk/auth

### Classes

- [Auth](#authclassesauthmd)
- [HTTPException](#authclasseshttpexceptionmd)

### Interfaces

- [AuthEventValueMap](#authinterfacesautheventvaluemapmd)

### Type Aliases

- [AuthFilters](#authtype-aliasesauthfiltersmd)


<a name="authclassesauthmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / Auth

## Class: Auth\<TExtra, TAuthReturn, TUser\>

Defined in: [src/auth/index.ts:11](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/index.ts#L11)

### Type Parameters

• **TExtra** = \{\}

• **TAuthReturn** *extends* `BaseAuthReturn` = `BaseAuthReturn`

• **TUser** *extends* `BaseUser` = `ToUserLike`\<`TAuthReturn`\>

### Constructors

#### new Auth()

> **new Auth**\<`TExtra`, `TAuthReturn`, `TUser`\>(): [`Auth`](#authclassesauthmd)\<`TExtra`, `TAuthReturn`, `TUser`\>

Defined in: [src/auth/index.ts:11](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/index.ts#L11)

##### Returns

[`Auth`](#authclassesauthmd)\<`TExtra`, `TAuthReturn`, `TUser`\>

### Methods

#### authenticate()

> **authenticate**\<`T`\>(`cb`): [`Auth`](#authclassesauthmd)\<`TExtra`, `T`\>

Defined in: [src/auth/index.ts:25](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/index.ts#L25)

##### Type Parameters

• **T** *extends* `BaseAuthReturn`

##### Parameters

###### cb

`AuthenticateCallback`\<`T`\>

##### Returns

[`Auth`](#authclassesauthmd)\<`TExtra`, `T`\>

***

#### on()

> **on**\<`T`\>(`event`, `callback`): `this`

Defined in: [src/auth/index.ts:32](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/index.ts#L32)

##### Type Parameters

• **T** *extends* `CallbackEvent`

##### Parameters

###### event

`T`

###### callback

`OnCallback`\<`T`, `TUser`\>

##### Returns

`this`


<a name="authclasseshttpexceptionmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / HTTPException

## Class: HTTPException

Defined in: [src/auth/error.ts:66](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L66)

### Extends

- `Error`

### Constructors

#### new HTTPException()

> **new HTTPException**(`status`, `options`?): [`HTTPException`](#authclasseshttpexceptionmd)

Defined in: [src/auth/error.ts:70](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L70)

##### Parameters

###### status

`number`

##### Returns

[`HTTPException`](#authclasseshttpexceptionmd)

##### Overrides

`Error.constructor`

### Properties

#### cause?

> `optional` **cause**: `unknown`

Defined in: node\_modules/typescript/lib/lib.es2022.error.d.ts:24

##### Inherited from

`Error.cause`

***

#### headers

> **headers**: `HeadersInit`

Defined in: [src/auth/error.ts:68](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L68)

***

#### message

> **message**: `string`

Defined in: node\_modules/typescript/lib/lib.es5.d.ts:1077

##### Inherited from

`Error.message`

***

#### name

> **name**: `string`

Defined in: node\_modules/typescript/lib/lib.es5.d.ts:1076

##### Inherited from

`Error.name`

***

#### stack?

> `optional` **stack**: `string`

Defined in: node\_modules/typescript/lib/lib.es5.d.ts:1078

##### Inherited from

`Error.stack`

***

#### status

> **status**: `number`

Defined in: [src/auth/error.ts:67](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L67)

***

#### prepareStackTrace()?

> `static` `optional` **prepareStackTrace**: (`err`, `stackTraces`) => `any`

Defined in: node\_modules/@types/node/globals.d.ts:28

Optional override for formatting stack traces

##### Parameters

###### err

`Error`

##### Returns

`any`

##### See

https://v8.dev/docs/stack-trace-api#customizing-stack-traces

##### Inherited from

`Error.prepareStackTrace`

***

#### stackTraceLimit

> `static` **stackTraceLimit**: `number`

Defined in: node\_modules/@types/node/globals.d.ts:30

##### Inherited from

`Error.stackTraceLimit`

### Methods

#### captureStackTrace()

> `static` **captureStackTrace**(`targetObject`, `constructorOpt`?): `void`

Defined in: node\_modules/@types/node/globals.d.ts:21

Create .stack property on a target object

##### Parameters

###### targetObject

`object`

##### Returns

`void`

##### Inherited from

`Error.captureStackTrace`


<a name="authinterfacesautheventvaluemapmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / AuthEventValueMap

## Interface: AuthEventValueMap

Defined in: [src/auth/types.ts:218](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L218)

### Properties

#### assistants:create

> **assistants:create**: `object`

Defined in: [src/auth/types.ts:226](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L226)

##### assistant\_id?

> `optional` **assistant\_id**: `Maybe`\<`string`\>

##### config?

> `optional` **config**: `Maybe`\<`AssistantConfig`\>

##### graph\_id

> **graph\_id**: `string`

##### if\_exists?

> `optional` **if\_exists**: `Maybe`\<`"raise"` \| `"do_nothing"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### name?

> `optional` **name**: `Maybe`\<`string`\>

***

#### assistants:delete

> **assistants:delete**: `object`

Defined in: [src/auth/types.ts:229](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L229)

##### assistant\_id

> **assistant\_id**: `string`

***

#### assistants:read

> **assistants:read**: `object`

Defined in: [src/auth/types.ts:227](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L227)

##### assistant\_id

> **assistant\_id**: `string`

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

***

#### assistants:search

> **assistants:search**: `object`

Defined in: [src/auth/types.ts:230](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L230)

##### graph\_id?

> `optional` **graph\_id**: `Maybe`\<`string`\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

***

#### assistants:update

> **assistants:update**: `object`

Defined in: [src/auth/types.ts:228](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L228)

##### assistant\_id

> **assistant\_id**: `string`

##### config?

> `optional` **config**: `Maybe`\<`AssistantConfig`\>

##### graph\_id?

> `optional` **graph\_id**: `Maybe`\<`string`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### name?

> `optional` **name**: `Maybe`\<`string`\>

##### version?

> `optional` **version**: `Maybe`\<`number`\>

***

#### crons:create

> **crons:create**: `object`

Defined in: [src/auth/types.ts:232](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L232)

##### cron\_id?

> `optional` **cron\_id**: `Maybe`\<`string`\>

##### end\_time?

> `optional` **end\_time**: `Maybe`\<`string`\>

##### payload?

> `optional` **payload**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### schedule

> **schedule**: `string`

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

##### user\_id?

> `optional` **user\_id**: `Maybe`\<`string`\>

***

#### crons:delete

> **crons:delete**: `object`

Defined in: [src/auth/types.ts:235](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L235)

##### cron\_id

> **cron\_id**: `string`

***

#### crons:read

> **crons:read**: `object`

Defined in: [src/auth/types.ts:233](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L233)

##### cron\_id

> **cron\_id**: `string`

***

#### crons:search

> **crons:search**: `object`

Defined in: [src/auth/types.ts:236](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L236)

##### assistant\_id?

> `optional` **assistant\_id**: `Maybe`\<`string`\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### crons:update

> **crons:update**: `object`

Defined in: [src/auth/types.ts:234](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L234)

##### cron\_id

> **cron\_id**: `string`

##### payload?

> `optional` **payload**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### schedule?

> `optional` **schedule**: `Maybe`\<`string`\>

***

#### store:delete

> **store:delete**: `object`

Defined in: [src/auth/types.ts:242](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L242)

##### key

> **key**: `string`

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

***

#### store:get

> **store:get**: `object`

Defined in: [src/auth/types.ts:239](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L239)

##### key

> **key**: `string`

##### namespace

> **namespace**: `Maybe`\<`string`[]\>

***

#### store:list\_namespaces

> **store:list\_namespaces**: `object`

Defined in: [src/auth/types.ts:241](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L241)

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### max\_depth?

> `optional` **max\_depth**: `Maybe`\<`number`\>

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### suffix?

> `optional` **suffix**: `Maybe`\<`string`[]\>

***

#### store:put

> **store:put**: `object`

Defined in: [src/auth/types.ts:238](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L238)

##### key

> **key**: `string`

##### namespace

> **namespace**: `string`[]

##### value

> **value**: `Record`\<`string`, `unknown`\>

***

#### store:search

> **store:search**: `object`

Defined in: [src/auth/types.ts:240](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L240)

##### filter?

> `optional` **filter**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### query?

> `optional` **query**: `Maybe`\<`string`\>

***

#### threads:create

> **threads:create**: `object`

Defined in: [src/auth/types.ts:219](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L219)

##### if\_exists?

> `optional` **if\_exists**: `Maybe`\<`"raise"` \| `"do_nothing"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:create\_run

> **threads:create\_run**: `object`

Defined in: [src/auth/types.ts:224](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L224)

##### after\_seconds?

> `optional` **after\_seconds**: `Maybe`\<`number`\>

##### assistant\_id

> **assistant\_id**: `string`

##### if\_not\_exists?

> `optional` **if\_not\_exists**: `Maybe`\<`"reject"` \| `"create"`\>

##### kwargs

> **kwargs**: `Record`\<`string`, `unknown`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### multitask\_strategy?

> `optional` **multitask\_strategy**: `Maybe`\<`"reject"` \| `"interrupt"` \| `"rollback"` \| `"enqueue"`\>

##### prevent\_insert\_if\_inflight?

> `optional` **prevent\_insert\_if\_inflight**: `Maybe`\<`boolean`\>

##### run\_id

> **run\_id**: `string`

##### status

> **status**: `Maybe`\<`"pending"` \| `"running"` \| `"error"` \| `"success"` \| `"timeout"` \| `"interrupted"`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:delete

> **threads:delete**: `object`

Defined in: [src/auth/types.ts:222](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L222)

##### run\_id?

> `optional` **run\_id**: `Maybe`\<`string`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:read

> **threads:read**: `object`

Defined in: [src/auth/types.ts:220](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L220)

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:search

> **threads:search**: `object`

Defined in: [src/auth/types.ts:223](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L223)

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### status?

> `optional` **status**: `Maybe`\<`"error"` \| `"interrupted"` \| `"idle"` \| `"busy"` \| `string` & `object`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

##### values?

> `optional` **values**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

***

#### threads:update

> **threads:update**: `object`

Defined in: [src/auth/types.ts:221](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L221)

##### action?

> `optional` **action**: `Maybe`\<`"interrupt"` \| `"rollback"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>


<a name="authtype-aliasesauthfiltersmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / AuthFilters

## Type Alias: AuthFilters\<TKey\>

> **AuthFilters**\<`TKey`\>: \{ \[key in TKey\]: string \| \{ \[op in "$contains" \| "$eq"\]?: string \} \}

Defined in: [src/auth/types.ts:367](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L367)

### Type Parameters

• **TKey** *extends* `string` \| `number` \| `symbol`


<a name="classesassistantsclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / AssistantsClient

## Class: AssistantsClient

Defined in: [client.ts:294](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L294)

### Extends

- `BaseClient`

### Constructors

#### new AssistantsClient()

> **new AssistantsClient**(`config`?): [`AssistantsClient`](#classesassistantsclientmd)

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`AssistantsClient`](#classesassistantsclientmd)

##### Inherited from

`BaseClient.constructor`

### Methods

#### create()

> **create**(`payload`): `Promise`\<`Assistant`\>

Defined in: [client.ts:359](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L359)

创建一个助手。

##### Parameters

###### payload

创建助手的载荷。

####### assistantId?

`string`

####### config?

`Config`

####### description?

`string`

####### graphId

`string`

####### ifExists?

`OnConflictBehavior`

####### metadata?

`Metadata`

####### name?

`string`

##### Returns

`Promise`\<`Assistant`\>

创建的助手。

***

#### delete()

> **delete**(`assistantId`): `Promise`\<`void`\>

Defined in: [client.ts:415](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L415)

删除助手。

##### Parameters

###### assistantId

`string`

助手的 ID。

##### Returns

`Promise`\<`void`\>

***

#### get()

> **get**(`assistantId`): `Promise`\<`Assistant`\>

Defined in: [client.ts:301](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L301)

按 ID 获取助手。

##### Parameters

###### assistantId

`string`

助手的 ID。

##### Returns

`Promise`\<`Assistant`\>

助手

***

#### getGraph()

> **getGraph**(`assistantId`, `options`?): `Promise`\<`AssistantGraph`\>

Defined in: [client.ts:311](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L311)

获取分配给可运行对象的图形的 JSON 表示。

##### Parameters

###### assistantId

`string`

助手的 ID。

###### options?

####### xray?

`number` \| `boolean`

是否在序列化图形表示中包含子图形。如果提供整数值，则仅包含深度小于或等于该值的子图形。

##### Returns

`Promise`\<`AssistantGraph`\>

序列化的图形

***

#### getSchemas()

> **getSchemas**(`assistantId`): `Promise`\<`GraphSchema`\>

Defined in: [client.ts:325](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L325)

获取分配给可运行对象的图形的状态和配置模式。

##### Parameters

###### assistantId

`string`

助手的 ID。

##### Returns

`Promise`\<`GraphSchema`\>

图形模式

***

#### getSubgraphs()

> **getSubgraphs**(`assistantId`, `options`?): `Promise`\<`Subgraphs`\>

Defined in: [client.ts:336](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L336)

按 ID 获取助手的子图形模式。

##### Parameters

###### assistantId

`string`

要获取其模式的助手的 ID。

###### options?

获取子图形的其他选项，例如命名空间或递归提取。

####### namespace?

`string`

####### recurse?

`boolean`

##### Returns

`Promise`\<`Subgraphs`\>

助手的子图形。

***

#### getVersions()

> **getVersions**(`assistantId`, `payload`?): `Promise`\<`AssistantVersion`[]\>

Defined in: [client.ts:453](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L453)

列出助手的全部版本。

##### Parameters

###### assistantId

`string`

助手的 ID。

###### payload?

####### limit?

`number`

####### metadata?

`Metadata`

####### offset?

`number`

##### Returns

`Promise`\<`AssistantVersion`[]\>

助手版本列表。

***

#### search()

> **search**(`query`?): `Promise`\<`Assistant`[]\>

Defined in: [client.ts:426](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L426)

列出助手。

##### Parameters

###### query?

查询选项。

####### graphId?

`string`

####### limit?

`number`

####### metadata?

`Metadata`

####### offset?

`number`

####### sortBy?

`AssistantSortBy`

####### sortOrder?

`SortOrder`

##### Returns

`Promise`\<`Assistant`[]\>

助手列表。

***

#### setLatest()

> **setLatest**(`assistantId`, `version`): `Promise`\<`Assistant`\>

Defined in: [client.ts:481](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L481)

更改助手的版本。

##### Parameters

###### assistantId

`string`

助手的 ID。

###### version

`number`

要更改到的版本。

##### Returns

`Promise`\<`Assistant`\>

更新后的助手。

***

#### update()

> **update**(`assistantId`, `payload`): `Promise`\<`Assistant`\>

Defined in: [client.ts:388](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L388)

更新助手。

##### Parameters

###### assistantId

`string`

助手的 ID。

###### payload

更新助手的载荷。

####### config?

`Config`

####### description?

`string`

####### graphId?

`string`

####### metadata?

`Metadata`

####### name?

`string`

##### Returns

`Promise`\<`Assistant`\>

更新后的助手。


<a name="classesclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / Client

## Class: Client\<TStateType, TUpdateType, TCustomEventType\>

Defined in: [client.ts:1448](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1448)

### Type Parameters

• **TStateType** = `DefaultValues`

• **TUpdateType** = `TStateType`

• **TCustomEventType** = `unknown`

### Constructors

#### new Client()

> **new Client**\<`TStateType`, `TUpdateType`, `TCustomEventType`\>(`config`?): [`Client`](#classesclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:1484](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1484)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`Client`](#classesclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

### Properties

#### ~ui

> **~ui**: `UiClient`

Defined in: [client.ts:1482](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1482)

**`Internal`**

用于与 UI 交互的客户端。
由 LoadExternalComponent 和 API 使用，未来可能会发生变化。

***

#### assistants

> **assistants**: [`AssistantsClient`](#classesassistantsclientmd)

Defined in: [client.ts:1456](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1456)

用于与助手交互的客户端。

***

#### crons

> **crons**: [`CronsClient`](#classescronsclientmd)

Defined in: [client.ts:1471](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1471)

用于与 cron 任务交互的客户端。

***

#### runs

> **runs**: [`RunsClient`](#classesrunsclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:1466](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1466)

用于与运行交互的客户端。

***

#### store

> **store**: [`StoreClient`](#classesstoreclientmd)

Defined in: [client.ts:1476](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1476)

用于与 KV 存储交互的客户端。

***

#### threads

> **threads**: [`ThreadsClient`](#classesthreadsclientmd)\<`TStateType`, `TUpdateType`\>

Defined in: [client.ts:1461](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1461)

用于与线程交互的客户端。


<a name="classescronsclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / CronsClient

## Class: CronsClient

Defined in: [client.ts:197](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L197)

### Extends

- `BaseClient`

### Constructors

#### new CronsClient()

> **new CronsClient**(`config`?): [`CronsClient`](#classescronsclientmd)

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`CronsClient`](#classescronsclientmd)

##### Inherited from

`BaseClient.constructor`

### Methods

#### create()

> **create**(`assistantId`, `payload`?): `Promise`\<`CronCreateResponse`\>

Defined in: [client.ts:238](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L238)

##### Parameters

###### assistantId

`string`

将用于此 cron 作业的助手 ID。

###### payload?

`CronsCreatePayload`

创建 cron 作业的载荷。

##### Returns

`Promise`\<`CronCreateResponse`\>

***

#### createForThread()

> **createForThread**(`threadId`, `assistantId`, `payload`?): `Promise`\<`CronCreateForThreadResponse`\>

Defined in: [client.ts:205](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L205)

##### Parameters

###### threadId

`string`

线程的 ID。

###### assistantId

`string`

将用于此 cron 作业的助手 ID。

###### payload?

`CronsCreatePayload`

创建 cron 作业的载荷。

##### Returns

`Promise`\<`CronCreateForThreadResponse`\>

创建的后台运行。

***

#### delete()

> **delete**(`cronId`): `Promise`\<`void`\>

Defined in: [client.ts:265](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L265)

##### Parameters

###### cronId

`string`

要删除的 Cron 作业的 Cron ID。

##### Returns

`Promise`\<`void`\>

***

#### search()

> **search**(`query`?): `Promise`\<`Cron`[]\>

Defined in: [client.ts:276](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L276)

##### Parameters

###### query?

查询选项。

####### assistantId?

`string`

####### limit?

`number`

####### offset?

`number`

####### threadId?

`string`

##### Returns

`Promise`\<`Cron`[]\>

Cron 列表。


<a name="classesrunsclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / RunsClient

## Class: RunsClient\<TStateType, TUpdateType, TCustomEventType\>

Defined in: [client.ts:776](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L776)

### Extends

- `BaseClient`

### Type Parameters

• **TStateType** = `DefaultValues`

• **TUpdateType** = `TStateType`

• **TCustomEventType** = `unknown`

### Constructors

#### new RunsClient()

> **new RunsClient**\<`TStateType`, `TUpdateType`, `TCustomEventType`\>(`config`?): [`RunsClient`](#classesrunsclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`RunsClient`](#classesrunsclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

##### Inherited from

`BaseClient.constructor`

### Methods

#### cancel()

> **cancel**(`threadId`, `runId`, `wait`, `action`): `Promise`\<`void`\>

Defined in: [client.ts:1063](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1063)

取消运行。

##### Parameters

###### threadId

`string`

线程的 ID。

###### runId

`string`

运行的 ID。

###### wait

`boolean` = `false`

取消时是否阻塞

###### action

`CancelAction` = `"interrupt"`

取消运行时要执行的操作。可能的值为 `interrupt` 或 `rollback`。默认为 `interrupt`。

##### Returns

`Promise`\<`void`\>

***

#### create()

> **create**(`threadId`, `assistantId`, `payload`?): `Promise`\<`Run`\>

Defined in: [client.ts:885](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L885)

创建运行。

##### Parameters

###### threadId

`string`

线程的 ID。

###### assistantId

`string`

将用于此运行的助手 ID。

###### payload?

`RunsCreatePayload`

创建运行的载荷。

##### Returns

`Promise`\<`Run`\>

创建的运行。

***

#### createBatch()

> **createBatch**(`payloads`): `Promise`\<`Run`[]\>

Defined in: [client.ts:921](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L921)

创建一组无状态后台运行。

##### Parameters

###### payloads

`RunsCreatePayload` & `object`[]

用于创建运行的载荷数组。

##### Returns

`Promise`\<`Run`[]\>

已创建的运行数组。

***

#### delete()

> **delete**(`threadId`, `runId`): `Promise`\<`void`\>

Defined in: [client.ts:1157](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1157)

删除运行。

##### Parameters

###### threadId

`string`

线程的 ID。

###### runId

`string`

运行的 ID。

##### Returns

`Promise`\<`void`\>

***

#### get()

> **get**(`threadId`, `runId`): `Promise`\<`Run`\>

Defined in: [client.ts:1050](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1050)

按 ID 获取运行。

##### Parameters

###### threadId

`string`

线程的 ID。

###### runId

`string`

运行的 ID。

##### Returns

`Promise`\<`Run`\>

运行。

***

#### join()

> **join**(`threadId`, `runId`, `options`?): `Promise`\<`void`\>

Defined in: [client.ts:1085](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1085)

阻塞直到运行完成。

##### Parameters

###### threadId

`string`

线程的 ID。

###### runId

`string`

运行的 ID。

###### options?

####### signal?

`AbortSignal`

##### Returns

`Promise`\<`void`\>

***

#### joinStream()

> **joinStream**(`threadId`, `runId`, `options`?): `AsyncGenerator`\<\{ `data`: `any`; `event`: `StreamEvent`; \}\>

Defined in: [client.ts:1111](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1111)

实时流式传输运行输出，直到运行完成。
输出不会被缓冲，“所以任何在此调用之前生成的输出都不会在此处收到”。

##### Parameters

###### threadId

`string`

线程的 ID。

###### runId

`string`

运行的 ID。

###### options?

用于控制流行为的附加选项：
  - signal：可用于取消流请求的 AbortSignal
  - cancelOnDisconnect：如果为 true，当客户端断开流连接时自动取消运行
  - streamMode：控制从流中接收什么类型的事件（可以是单个模式或模式数组）
       必须是创建运行后启用的所有流模式的子集。后台运行默认启用所有流模式的并集。

`AbortSignal` | \{ `cancelOnDisconnect`: `boolean`; `signal`: `AbortSignal`; `streamMode`: `StreamMode` \| `StreamMode`[]; \}

##### Returns

`AsyncGenerator`\<\{ `data`: `any`; `event`: `StreamEvent`; \}\>

产生流部分的异步生成器。

***

#### list()

> **list**(`threadId`, `options`?): `Promise`\<`Run`[]\>

Defined in: [client.ts:1013](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1013)

列出线程的所有运行。

##### Parameters

###### threadId

`string`

线程的 ID。

###### options?

过滤和分页选项。

####### limit?

`number`

要返回的最大运行次数。
默认为 10

####### offset?

`number`

开始的偏移量。
默认为 0。

####### status?

`RunStatus`

要过滤的运行状态。

##### Returns

`Promise`\<`Run`[]\>

运行列表。

***

#### stream()

创建运行并流式传输结果。

##### Param

线程的 ID。

##### Param

将用于此运行的助手 ID。

##### Param

创建运行的载荷。

##### Call Signature

> **stream**\<`TStreamMode`, `TSubgraphs`\>(`threadId`, `assistantId`, `payload`?): `TypedAsyncGenerator`\<`TStreamMode`, `TSubgraphs`, `TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:781](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L781)

###### Type Parameters

• **TStreamMode** *extends* `StreamMode` \| `StreamMode`[] = `StreamMode`

• **TSubgraphs** *extends* `boolean` = `false`

###### Parameters

####### threadId

`null`

####### assistantId

`string`

####### payload?

`Omit`\<`RunsStreamPayload`\<`TStreamMode`, `TSubgraphs`\>, `"multitaskStrategy"` \| `"onCompletion"`\>

###### Returns

`TypedAsyncGenerator`\<`TStreamMode`, `TSubgraphs`, `TStateType`, `TUpdateType`, `TCustomEventType`\>

##### Call Signature

> **stream**\<`TStreamMode`, `TSubgraphs`\>(`threadId`, `assistantId`, `payload`?): `TypedAsyncGenerator`\<`TStreamMode`, `TSubgraphs`, `TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:799](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L799)

###### Type Parameters

• **TStreamMode** *extends* `StreamMode` \| `StreamMode`[] = `StreamMode`

• **TSubgraphs** *extends* `boolean` = `false`

###### Parameters

####### threadId

`string`

####### assistantId

`string`

####### payload?

`RunsStreamPayload`\<`TStreamMode`, `TSubgraphs`\>

###### Returns

`TypedAsyncGenerator`\<`TStreamMode`, `TSubgraphs`, `TStateType`, `TUpdateType`, `TCustomEventType`\>

***

#### wait()

创建运行并等待其完成。

##### Param

线程的 ID。

##### Param

将用于此运行的助手 ID。

##### Param

创建运行的载荷。

##### Call Signature

> **wait**(`threadId`, `assistantId`, `payload`?): `Promise`\<`DefaultValues`\>

Defined in: [client.ts:938](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L938)

###### Parameters

####### threadId

`null`

####### assistantId

`string`

####### payload?

`Omit`\<`RunsWaitPayload`, `"multitaskStrategy"` \| `"onCompletion"`\>

###### Returns

`Promise`\<`DefaultValues`\>

##### Call Signature

> **wait**(`threadId`, `assistantId`, `payload`?): `Promise`\<`DefaultValues`\>

Defined in: [client.ts:944](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L944)

###### Parameters

####### threadId

`string`

####### assistantId

`string`

####### payload?

`RunsWaitPayload`

###### Returns

`Promise`\<`DefaultValues`\>


<a name="classesstoreclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / StoreClient

## Class: StoreClient

Defined in: [client.ts:1175](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1175)

### Extends

- `BaseClient`

### Constructors

#### new StoreClient()

> **new StoreClient**(`config`?): [`StoreClient`](#classesstoreclientmd)

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`StoreClient`](#classesstoreclientmd)

##### Inherited from

`BaseClient.constructor`

### Methods

#### deleteItem()

> **deleteItem**(`namespace`, `key`): `Promise`\<`void`\>

Defined in: [client.ts:1296](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1296)

删除一个条目。

##### Parameters

###### namespace

`string`[]

表示命名空间路径的字符串列表。

###### key

`string`

条目的唯一标识符。

##### Returns

`Promise`\<`void`\>

Promise<void>

***

#### getItem()

> **getItem**(`namespace`, `key`, `options`?): `Promise`\<`null` \| `Item`\>

Defined in: [client.ts:1252](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1252)

检索单个条目。

##### Parameters

###### namespace

`string`[]

表示命名空间路径的字符串列表。

###### key

`string`

条目的唯一标识符。

###### options?

####### refreshTtl?

`null` \| `boolean`

是否在此读取操作上刷新 TTL。如果为 null，则使用存储的默认行为。

##### Returns

`Promise`\<`null` \| `Item`\>

Promise<Item>

##### Example

```typescript
const item = await client.store.getItem(
  ["documents", "user123"],
  "item456",
  { refreshTtl: true }
);
console.log(item);
// {
//   namespace: ["documents", "user123"],
//   key: "item456",
//   value: { title: "My Document", content: "Hello World" },
//   createdAt: "2024-07-30T12:00:00Z",
//   updatedAt: "2024-07-30T12:00:00Z"
// }
```

***

#### listNamespaces()

> **listNamespaces**(`options`?): `Promise`\<`ListNamespaceResponse`\>

Defined in: [client.ts:1392](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1392)

列出具有可选匹配条件的命名空间。

##### Parameters

###### options?

####### limit?

`number`

要返回的最大命名空间数量（默认为 100）。

####### maxDepth?

`number`

可选整数，指定要返回的命名空间的最大深度。

####### offset?

`number`

在返回结果之前要跳过的命名空间数量（默认为 0）。

####### prefix?

`string`[]

可选字符串列表，表示用于过滤命名空间的字符串。

####### suffix?

`string`[]

可选字符串列表，表示用于过滤命名空间的后缀。

##### Returns

`Promise`\<`ListNamespaceResponse`\>

Promise<ListNamespaceResponse>

***

#### putItem()

> **putItem**(`namespace`, `key`, `value`, `options`?): `Promise`\<`void`\>

Defined in: [client.ts:1196](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1196)

存储或更新条目。

##### Parameters

###### namespace

`string`[]

表示命名空间路径的字符串列表。

###### key

`string`

命名空间内条目的唯一标识符。

###### value

`Record`\<`string`, `any`\>

包含条目数据的字典。

###### options?

####### index?

`null` \| `false` \| `string`[]

控制搜索索引 - null（使用默认值），false（禁用），或要索引的字段路径列表。

####### ttl?

`null` \| `number`

条目的生存时间（以分钟为单位），或 null 表示无过期。

##### Returns

`Promise`\<`void`\>

Promise<void>

##### Example

```typescript
await client.store.putItem(
  ["documents", "user123"],
  "item456",
  { title: "My Document", content: "Hello World" },
  { ttl: 60 } // expires in 60 minutes
);
```

***

#### searchItems()

> **searchItems**(`namespacePrefix`, `options`?): `Promise`\<`SearchItemsResponse`\>

Defined in: [client.ts:1347](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1347)

在命名空间前缀内搜索条目。

##### Parameters

###### namespacePrefix

`string`[]

表示命名空间前缀的字符串列表。

###### options?

####### filter?

`Record`\<`string`, `any`\>

用于过滤结果的可选键值对字典。

####### limit?

`number`

要返回的最大条目数（默认为 10）。

####### offset?

`number`

在返回结果之前要跳过的条目数（默认为 0）。

####### query?

`string`

可选搜索查询。

####### refreshTtl?

`null` \| `boolean`

是否刷新此搜索返回的条目的 TTL。如果为 null，则使用存储的默认行为。

##### Returns

`Promise`\<`SearchItemsResponse`\>

Promise<SearchItemsResponse>

##### Example

```typescript
const results = await client.store.searchItems(
  ["documents"],
  {
    filter: { author: "John Doe" },
    limit: 5,
    refreshTtl: true
  }
);
console.log(results);
// {
//   items: [
//     {
//       namespace: ["documents", "user123"],
//       key: "item789",
//       value: { title: "Another Document", author: "John Doe" },
//       createdAt: "2024-07-30T12:00:00Z",
//       updatedAt: "2024-07-30T12:00:00Z"
//     },
//     // ... additional items ...
//   ]
// }
```


<a name="classesthreadsclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / ThreadsClient

## Class: ThreadsClient\<TStateType, TUpdateType\>

Defined in: [client.ts:489](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L489)

### Extends

- `BaseClient`

### Type Parameters

• **TStateType** = `DefaultValues`

• **TUpdateType** = `TStateType`

### Constructors

#### new ThreadsClient()

> **new ThreadsClient**\<`TStateType`, `TUpdateType`\>(`config`?): [`ThreadsClient`](#classesthreadsclientmd)\<`TStateType`, `TUpdateType`\>

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`ThreadsClient`](#classesthreadsclientmd)\<`TStateType`, `TUpdateType`\>

##### Inherited from

`BaseClient.constructor`

### Methods

#### copy()

> **copy**(`threadId`): `Promise`\<`Thread`\<`TStateType`\>\>

Defined in: [client.ts:566](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L566)

复制现有线程。

##### Parameters

###### threadId

`string`

要复制的线程 ID。

##### Returns

`Promise`\<`Thread`\<`TStateType`\>\>

新复制的线程。

***

#### create()

> **create**(`payload`?): `Promise`\<`Thread`\<`TStateType`\>\>

Defined in: [client.ts:511](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L511)

创建一个新线程。

##### Parameters

###### payload?

创建线程的载荷。

####### graphId?

`string`

要与线程关联的 Graph ID。

####### ifExists?

`OnConflictBehavior`

如何处理重复创建。

**`Default`**

```ts
"raise"
```

####### metadata?

`Metadata`

线程的元数据。

####### supersteps?

`object`[]

在创建线程时应用一组 supersteps，每个 superstep 包含一系列更新。
用于在部署之间复制线程。

####### threadId?

`string`

要创建的线程 ID。
如果未提供，将生成随机 UUID。

##### Returns

`Promise`\<`Thread`\<`TStateType`\>\>

创建的线程。

***

#### delete()

> **delete**(`threadId`): `Promise`\<`void`\>

Defined in: [client.ts:599](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L599)

删除线程。

##### Parameters

###### threadId

`string`

线程 ID。

##### Returns

`Promise`\<`void`\>

***

#### get()

> **get**\<`ValuesType`\>(`threadId`): `Promise`\<`Thread`\<`ValuesType`\>\>

Defined in: [client.ts:499](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L499)

按 ID 获取线程。

##### Type Parameters

• **ValuesType** = `TStateType`

##### Parameters

###### threadId

`string`

线程 ID。

##### Returns

`Promise`\<`Thread`\<`ValuesType`\>\>

线程。

***

#### getHistory()

> **getHistory**\<`ValuesType`\>(`threadId`, `options`?): `Promise`\<`ThreadState`\<`ValuesType`\>[]\>

Defined in: [client.ts:752](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L752)

获取线程的所有 past states。

##### Type Parameters

• **ValuesType** = `TStateType`

##### Parameters

###### threadId

`string`

线程 ID。

###### options?

附加选项。

####### before?

`Config`

####### checkpoint?

`Partial`\<`Omit`\<`Checkpoint`, `"thread_id"`\>\>

####### limit?

`number`

####### metadata?

`Metadata`

##### Returns

`Promise`\<`ThreadState`\<`ValuesType`\>[]\>

线程状态列表。

***

#### getState()

> **getState**\<`ValuesType`\>(`threadId`, `checkpoint`?, `options`?): `Promise`\<`ThreadState`\<`ValuesType`\>\>

Defined in: [client.ts:659](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L659)

获取线程的状态。

##### Type Parameters

• **ValuesType** = `TStateType`

##### Parameters

###### threadId

`string`

线程 ID。

###### checkpoint?

`string` | `Checkpoint`

###### options?

####### subgraphs?

`boolean`

##### Returns

`Promise`\<`ThreadState`\<`ValuesType`\>\>

线程状态。

***

#### patchState()

> **patchState**(`threadIdOrConfig`, `metadata`): `Promise`\<`void`\>

Defined in: [client.ts:722](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L722)

更新线程的元数据。

##### Parameters

###### threadIdOrConfig

要更新状态的线程 ID 或配置。

`string` | `Config`

###### metadata

要用作状态更新的元数据。

##### Returns

`Promise`\<`void`\>

***

#### search()

> **search**\<`ValuesType`\>(`query`?): `Promise`\<`Thread`\<`ValuesType`\>[]\>

Defined in: [client.ts:611](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L611)

列出线程。

##### Type Parameters

• **ValuesType** = `TStateType`

##### Parameters

###### query?

查询选项

####### limit?

`number`

要返回的最大线程数。
默认为 10

####### metadata?

`Metadata`

用于过滤线程的元数据。

####### offset?

`number`

开始的偏移量。

####### sortBy?

`ThreadSortBy`

排序依据。

####### sortOrder?

`SortOrder`

排序顺序。
必须是 'asc' 或 'desc' 之一。

####### status?

`ThreadStatus`

要过滤的线程状态。
必须是 'idle'、'busy'、'interrupted' 或 'error' 之一。

##### Returns

`Promise`\<`Thread`\<`ValuesType`\>[]\>

线程列表。

***

#### update()

> **update**(`threadId`, `payload`?): `Promise`\<`Thread`\<`DefaultValues`\>\>

Defined in: [client.ts:579](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L579)

更新线程。

##### Parameters

###### threadId

`string`

线程 ID。

###### payload?

更新线程的载荷。

####### metadata?

`Metadata`

线程的元数据。

##### Returns

`Promise`\<`Thread`\<`DefaultValues`\>\>

更新后的线程。

***

#### updateState()

> **updateState**\<`ValuesType`\>(`threadId`, `options`): `Promise`\<`Pick`\<`Config`, `"configurable"`\>\>

Defined in: [client.ts:693](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L693)

向线程添加状态。

##### Type Parameters

• **ValuesType** = `TUpdateType`

##### Parameters

###### threadId

`string`

线程的 ID。

###### options

####### asNode?

`string`

####### checkpoint?

`Checkpoint`

####### checkpointId?

`string`

####### values

`ValuesType`

##### Returns

`Promise`\<`Pick`\<`Config`, `"configurable"`\>\>


<a name="functionsgetapikeymd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / getApiKey

## Function: getApiKey()

> **getApiKey**(`apiKey`?): `undefined` \| `string`

Defined in: [client.ts:53](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L53)

从环境变量中获取 API 密钥。
优先级：
  1. 显式参数
  2. LANGGRAPH_API_KEY
  3. LANGSMITH_API_KEY
  4. LANGCHAIN_API_KEY

### Parameters

#### apiKey?

`string`

作为参数提供的可选 API 密钥。

### Returns

`undefined` \| `string`

如果找到 API 密钥，则返回该密钥，否则返回 undefined。


<a name="interfacesclientconfigmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / ClientConfig

## Interface: ClientConfig

Defined in: [client.ts:71](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L71)

### Properties

#### apiKey?

> `optional` **apiKey**: `string`

Defined in: [client.ts:73](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L73)

***

#### apiUrl?

> `optional` **apiUrl**: `string`

Defined in: [client.ts:72](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L72)

***

#### callerOptions?

> `optional` **callerOptions**: `AsyncCallerParams`

Defined in: [client.ts:74](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L74)

***

#### defaultHeaders?

> `optional` **defaultHeaders**: `Record`\<`string`, `undefined` \| `null` \| `string`\>

Defined in: [client.ts:76](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L76)

***

#### timeoutMs?

> `optional` **timeoutMs**: `number`

Defined in: [client.ts:75](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L75)


<a name="reactreadmemd"></a>

**@langchain/langgraph-sdk**

***

## @langchain/langgraph-sdk/react

### Interfaces

- [UseStream](#reactinterfacesusestreammd)
- [UseStreamOptions](#reactinterfacesusestreamoptionsmd)

### Type Aliases

- [MessageMetadata](#reacttype-aliasesmessagemetadatamd)

### Functions

- [useStream](#reactfunctionsusestreammd)


<a name="reactfunctionsusestreammd"></a>

[**@langchain/langgraph-sdk**](#reactreadmemd)

***

[@langchain/langgraph-sdk](#reactreadmemd) / useStream

## Function: useStream()

> **useStream**\<`StateType`, `Bag`\>(`options`): [`UseStream`](#reactinterfacesusestreammd)\<`StateType`, `Bag`\>

Defined in: [react/stream.tsx:618](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L618)

### Type Parameters

• **StateType** *extends* `Record`\<`string`, `unknown`\> = `Record`\<`string`, `unknown`\>

• **Bag** *extends* `object` = `BagTemplate`

### Parameters

#### options

[`UseStreamOptions`](#reactinterfacesusestreamoptionsmd)\<`StateType`, `Bag`\>

### Returns

[`UseStream`](#reactinterfacesusestreammd)\<`StateType`, `Bag`\>


<a name="reactinterfacesusestreammd"></a>

[**@langchain/langgraph-sdk**](#reactreadmemd)

***

[@langchain/langgraph-sdk](#reactreadmemd) / UseStream

## Interface: UseStream\<StateType, Bag\>

Defined in: [react/stream.tsx:507](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L507)

### Type Parameters

• **StateType** *extends* `Record`\<`string`, `unknown`\> = `Record`\<`string`, `unknown`\>

• **Bag** *extends* `BagTemplate` = `BagTemplate`

### Properties

#### assistantId

> **assistantId**: `string`

Defined in: [react/stream.tsx:592](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L592)

要使用的助手 ID。

***

#### branch

> **branch**: `string`

Defined in: [react/stream.tsx:542](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L542)

线程的当前分支。

***

#### client

> **client**: `Client`

Defined in: [react/stream.tsx:587](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L587)

用于发送请求和接收响应的 LangGraph SDK 客户端。

***

#### error

> **error**: `unknown`

Defined in: [react/stream.tsx:519](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L519)

最后看到的来自线程或流式传输中的错误。

***

#### experimental\_branchTree

> **experimental\_branchTree**: `Sequence`\<`StateType`\>

Defined in: [react/stream.tsx:558](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L558)

**`Experimental`**

线程的所有分支树。

***

#### getMessagesMetadata()

> **getMessagesMetadata**: (`message`, `index`?) => `undefined` \| [`MessageMetadata`](#reacttype-aliasesmessagemetadatamd)\<`StateType`\>

Defined in: [react/stream.tsx:579](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L579)

获取消息的元数据，例如消息首次出现的线程状态和分支信息。

##### Parameters

###### message

`Message`

要获取其元数据的消息。

###### index?

`number`

消息在线程中的索引。

##### Returns

`undefined` \| [`MessageMetadata`](#reacttype-aliasesmessagemetadatamd)\<`StateType`\>

消息的元数据。

***

#### history

> **history**: `ThreadState`\<`StateType`\>[]

Defined in: [react/stream.tsx:552](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L552)

线程状态的历史记录（扁平化）。

***

#### interrupt

> **interrupt**: `undefined` \| `Interrupt`\<`GetInterruptType`\<`Bag`\>\>

Defined in: [react/stream.tsx:563](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L563)

如果线程被中断，则获取流的中断值。

***

#### isLoading

> **isLoading**: `boolean`

Defined in: [react/stream.tsx:524](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L524)

流是否正在运行。

***

#### messages

> **messages**: `Message`[]

Defined in: [react/stream.tsx:569](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L569)

从线程推断出的消息。
将自动更新接收到的消息块。

***

#### setBranch()

> **setBranch**: (`branch`) => `void`

Defined in: [react/stream.tsx:547](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L547)

设置线程的分支。

##### Parameters

###### branch

`string`

##### Returns

`void`

***

#### stop()

> **stop**: () => `void`

Defined in: [react/stream.tsx:529](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L529)

停止流。

##### Returns

`void`

***

#### submit()

> **submit**: (`values`, `options`?) => `void`

Defined in: [react/stream.tsx:534](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L534)

创建并流式传输线程的运行。

##### Parameters

###### values

`undefined` | `null` | `GetUpdateType`\<`Bag`, `StateType`\>

###### options?

`SubmitOptions`\<`StateType`, `GetConfigurableType`\<`Bag`\>\>

##### Returns

`void`

***

#### values

> **values**: `StateType`

Defined in: [react/stream.tsx:514](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L514)

线程的当前值。


<a name="reactinterfacesusestreamoptionsmd"></a>

[**@langchain/langgraph-sdk**](#reactreadmemd)

***

[@langchain/langgraph-sdk](#reactreadmemd) / UseStreamOptions

## Interface: UseStreamOptions\<StateType, Bag\>

Defined in: [react/stream.tsx:408](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L408)

### Type Parameters

• **StateType** *extends* `Record`\<`string`, `unknown`\> = `Record`\<`string`, `unknown`\>

• **Bag** *extends* `BagTemplate` = `BagTemplate`

### Properties

#### apiKey?

> `optional` **apiKey**: `string`

Defined in: [react/stream.tsx:430](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L430)

要使用的 API 密钥。

***

#### apiUrl?

> `optional` **apiUrl**: `string`

Defined in: [react/stream.tsx:425](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L425)

要使用的 API 的 URL。

***

#### assistantId

> **assistantId**: `string`

Defined in: [react/stream.tsx:415](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L415)

要使用的助手 ID。

***

#### callerOptions?

> `optional` **callerOptions**: `AsyncCallerParams`

Defined in: [react/stream.tsx:435](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L435)

自定义调用选项，例如自定义 fetch 实现。

***

#### client?

> `optional` **client**: `Client`\<`DefaultValues`, `DefaultValues`, `unknown`\>

Defined in: [react/stream.tsx:420](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L420)

用于发送请求的客户端。

***

#### defaultHeaders?

> `optional` **defaultHeaders**: `Record`\<`string`, `undefined` \| `null` \| `string`\>

Defined in: [react/stream.tsx:440](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L440)

随请求发送的默认标头。

***

#### messagesKey?

> `optional` **messagesKey**: `string`

Defined in: [react/stream.tsx:448](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L448)

指定包含消息的状态中的键。
默认为 "messages"。

##### Default

```ts
"messages"
```

***

#### onCustomEvent()?

> `optional` **onCustomEvent**: (`data`, `options`) => `void`

Defined in: [react/stream.tsx:470](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L470)

接收到自定义事件时调用。

##### Parameters

###### data

`GetCustomEventType`\<`Bag`\>

###### options

####### mutate

(`update`) => `void`

##### Returns

`void`

***

#### onDebugEvent()?

> `optional` **onDebugEvent**: (`data`) => `void`

Defined in: [react/stream.tsx:494](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L494)

**`Internal`**

接收到调试事件时调用。
此 API 是实验性的，可能会发生更改。

##### Parameters

###### data

`unknown`

##### Returns

`void`

***

#### onError()?

> `optional` **onError**: (`error`) => `void`

Defined in: [react/stream.tsx:453](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L453)

发生错误时调用。

##### Parameters

###### error

`unknown`

##### Returns

`void`

***

#### onFinish()?

> `optional` **onFinish**: (`state`) => `void`

Defined in: [react/stream.tsx:458](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L458)

流完成时调用。

##### Parameters

###### state

`ThreadState`\<`StateType`\>

##### Returns

`void`

***

#### onLangChainEvent()?

> `optional` **onLangChainEvent**: (`data`) => `void`

Defined in: [react/stream.tsx:488](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L488)

接收到 LangChain 事件时调用。

##### Parameters

###### data

####### data

`unknown`

####### event

`string` & `object` \| `"on_tool_start"` \| `"on_tool_stream"` \| `"on_tool_end"` \| `"on_chat_model_start"` \| `"on_chat_model_stream"` \| `"on_chat_model_end"` \| `"on_llm_start"` \| `"on_llm_stream"` \| `"on_llm_end"` \| `"on_chain_start"` \| `"on_chain_stream"` \| `"on_chain_end"` \| `"on_retriever_start"` \| `"on_retriever_stream"` \| `"on_retriever_end"` \| `"on_prompt_start"` \| `"on_prompt_stream"` \| `"on_prompt_end"`

####### metadata

`Record`\<`string`, `unknown`\>

####### name

`string`

####### parent_ids

`string`[]

####### run_id

`string`

####### tags

`string`[]

##### Returns

`void`

##### See

https://langchain-ai.github.io/langgraph/cloud/how-tos/stream_events/#stream-graph-in-events-mode 以获取更多详细信息。

***

#### onMetadataEvent()?

> `optional` **onMetadataEvent**: (`data`) => `void`

Defined in: [react/stream.tsx:482](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L482)

接收到元数据事件时调用。

##### Parameters

###### data

####### run_id

`string`

####### thread_id

`string`

##### Returns

`void`

***

#### onThreadId()?

> `optional` **onThreadId**: (`threadId`) => `void`

Defined in: [react/stream.tsx:504](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L504)

线程 ID 更新时调用（例如，创建新线程时）。

##### Parameters

###### threadId

`string`

##### Returns

`void`

***

#### onUpdateEvent()?

> `optional` **onUpdateEvent**: (`data`) => `void`

Defined in: [react/stream.tsx:463](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L463)

接收到更新事件时调用。

##### Parameters

###### data

##### Returns

`void`

***

#### threadId?

> `optional` **threadId**: `null` \| `string`

Defined in: [react/stream.tsx:499](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L499)

用于获取历史记录和当前值的线程 ID。


<a name="reacttype-aliasesmessagemetadatamd"></a>

[**@langchain/langgraph-sdk**](#reactreadmemd)

***

[@langchain/langgraph-sdk](#reactreadmemd) / MessageMetadata

## Type Alias: MessageMetadata\<StateType\>

> **MessageMetadata**\<`StateType`\>: `object`

Defined in: [react/stream.tsx:169](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L169)

### Type Parameters

• **StateType** *extends* `Record`\<`string`, `unknown`\>

### Type declaration

#### branch

> **branch**: `string` \| `undefined`

消息的分支。

#### branchOptions

> **branchOptions**: `string`[] \| `undefined`

该消息所属的分支列表。
这对于显示分支控件很有用。

#### firstSeenState

> **firstSeenState**: `ThreadState`\<`StateType`\> \| `undefined`

消息首次出现的线程状态。

#### messageId

> **messageId**: `string`

使用的消息 ID。