---
title: Data Validation with Typescript
date: 2020-09-16 22:04:29
categories:
- code
- backend
---

To build backend API, one of the problem is validate the external data. Because the incoming data is not necessary valid. The invalid data might come from user input, wrong call of API in frontend. It might also come from malicious attackers. Thus, validation in backend is neccessary no matter frontend did it or not.

Since I'm using Typescript to build backend in [Todo project](https://github.com/zlguo1996/Todo), it would reduce the redundancy of code if I can make use of existing type definition in type definition. [This post](https://2ality.com/2020/06/validating-data-typescript.html#approaches-for-data-validation-in-typescript) introduces several approaches for data validation categorized by using JSON schema or not. In my project, I tried the approach that converting Typescript types to JSON schema.

## Packages
- [typescript-json-schema](https://github.com/YousefED/typescript-json-schema): Generate json-schemas from Typescript sources.
- [ajv](https://ajv.js.org/): JSON schema validator

## Approach
1. Use `typescript-json-schema` to pre-compile existing type to JSON schema.
2. Use `ajv` to make use of the schema from `1` to validate the request data.

## Code
1. Typescript to JSON schema
    In my project, I store types used in Web API in [common folder](https://github.com/zlguo1996/Todo/tree/master/services/common) to use them both in frontend code and backend code.
    After write validation types in `src/apiTypes`, the following command in `package.json` is used to convert typescript to JSON schema file. (use `install` command name because it would be called after `npm install`. Please refer to [lifecycle scripts](https://docs.npmjs.com/misc/scripts))
    ```json
    "scripts": {
        "install": "typescript-json-schema src/apiTypes.ts * --noExtraProps --out src/schema.json"
    },
    ```
    The **benefit** of `typescript-json-schema` is that it allows using annotations to enhance the typescript properties in convertion. Like the following:
    ```js
    export interface TodoItem {
        /**
         * @minLength 0
        * @maxLength 100
        */
        text: string,
        state: TodoItemState,
    }
    ```
2. Validation with JSON schema
   After export JSON schema object from `common`, we can use it with `ajv`:
   ```js
   const ajv = new Ajv({
        schemaId: 'auto',
        allErrors: true,
    })
    ajv.addSchema(itemSchema, 'item')
   ```
   Then, the data could be validated like this:
   ```js
   const validator = ajv.getSchema('item#/definitions/AddItem')
   const result = validator(data)
   ```
   For details in the URI parameter of `ajv.getSchema` function, please refer to the [structuring complex schema](https://json-schema.org/understanding-json-schema/structuring.html)
