# nest-authz

A access control library for [NestJS](https://nestjs.com/) which built on [node-casbin](https://github.com/casbin/node-casbin).

Casbin is a powerful and efficient open-source access control library. It provides support for enforcing authorization based on various access control models like ACL, RBAC, ABAC. For detailed info, check out the [official docs](https://casbin.org/en/)

## How to use

### Installation

```bat
$ npm install --save nest-authz
```

### Define Access Control Model

Firstly, you should create your own casbin access control model. Checkout [related docs](https://github.com/casbin/node-casbin#supported-models) if you have not.

### Initialization

Register nest-authz with options in the AppModule as follows:

```
AuthZModule.register(options)
```

`options` is an object literal containing options.

- `model` (REQUIRED) is a path string to the casbin model.
- `policy` (REQUIRED) is a path string to the casbin policy file or adapter
- `usernameFromContext` (REQUIRED) is a function that accepts `ExecutionContext`(the param of guard method `canActivate`) as the only parameter and returns either the username as a string or null. The `AuthZGuard` uses username to determine user's permission internally.

An example configuration which reads username from the http request.

```typescript
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    AuthZModule.register({
      model: 'model.conf',
      policy: TypeORMAdapter.newAdapter({
        name: 'casbin',
        type: 'mysql',
        host: 'localhost',
        port: 3306,
        username: 'root',
        password: 'password',
        database: 'nestdb'
      }),
      usernameFromContext: (ctx) => {
        const request = ctx.switchToHttp().getRequest();
        return request.user && request.user.username;
      }
    }),
  ],
  controllers: [AppController],
  providers: [AppService]
})
```

### Checking Permissions

#### Using `@UsePermissions` Decorator

The `@UserPermissions` decorator is the easiest and most common way of checking permissions. Consider the method shown below:

```typescript
  @Get('users')
  @UseGuards(AuthZGuard)
  @UsePermissions({
    action: AuthActionVerb.READ,
    resource: 'USER',
    possession: AuthPossession.ANY
  })
  async findAllUsers() {}

```

The `findAllUsers` method can not be called by a user who is not granted the permission to read any user.

The value of property `resource` is a magic string just for demonstrating. In the real-world applications you should avoid magic strings. Resources should be kept in the separated file like `resources.ts`

The param of `UsePermissions` are some objects with required properties `action`、 `resource`、 `possession` and an optional `isOwn`.

- `action` is an enum value of `AuthActionVerb`.
- `resource` is a resource string the request is accessing.
- `possession` is an enum value of `AuthPossession`.
- `isOwn` is a function that accepts `ExecutionContext`(the param of guard method `canActivate`) as the only parameter and returns boolean. The `AuthZGuard` uses it to determine whether the user is the owner of the resource. A default `isOwn` function which returns `false` will be used if not defined.

You can define multiple permissions, but only when all of them satisfied, could you access the route. For example:

```
@UsePermissions({
  action: AuthActionVerb.READ,
  resource: 'USER_ADDRESS',
  possession: AuthPossession.ANY
}, {
  action; AuthActionVerb.READ,
  resource: 'USER_ROLES,
  possession: AuthPossession.ANY
})
```

Only when the user is granted both permissions of reading any user address and reading any roles, could he/she access the route.

#### Using `AuthzRBACService` or `AuthzManagementService`

While the `@UsePermissions` decorator is good enough for most cases, there are situations where we may want to check for a permission in a method's body. We can inject and use `AuthzRBACService` or `AuthzManagementService` which are wrappers of casbin api for that as shown in the example below:

```typescript
import { Controller, Get, UnauthorizedException } from '@nestjs/common';
import {
  AuthZGuard,
  AuthZRBACService,
  AuthActionVerb,
  AuthPossession,
  UsePermissions
} from 'nest-authz';

@Controller()
export class AppController {
  constructor(private readonly rbacSrv: AuthZRBACService) {}

  @Get('users')
  async findAllUsers() {
    const isPermitted = await this.rbacSrv.hasPermissionForUser();
    if (!isPermitted) {
      throw new UnauthorizedException(
        'You are not authorized to read users list'
      );
    }
    // A user can not reach this point if he/she is not granted for permission read users
  }
}
```

## Example

For more detailed information, checkout the working example in
directory `/example`

## License

This project is licensed under the MIT license.

## Contact

If you have any issues or feature requests, contact me. PR is welcomed.

- dreamdeviloo@163.com
