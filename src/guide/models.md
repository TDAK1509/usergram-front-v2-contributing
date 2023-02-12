# src/models

This directory manages models. This usually works together with [src/api](#srcapi).

## What we should do

### Use `class` instead of `interface` for model

```ts
// src/models/LoginUser.ts

export class LoginUser {
  constructor(
    public readonly userId: number, // <- must be camelCase
    public readonly username: string,
    public readonly role: string
  ) {}
}
```

#### Reason for using `class` instead of `interface`

It's not always easy to convert from `interface` to `class`. Let's say we have interface `LoginUser` and after a while, we need to add 3 getters for it.

```ts
// models/LoginUser.ts
export interface LoginUser {
  userId: number;
  username: string;
  role: string;
}

// In this case these 3 functions actually can be combined into one,
// But sometimes we cannot combine.
export const isAdmin = (role: string) => role === "admin";
export const isStaff = (role: string) => role === "staff";
export const isGuest = (role: string) => role === "guest";
```

```ts
// MyUser.vue
import { isAdmin, isStaf, isGuest, LoginUser } from "@/models/LoginUser";

const myUser: LoginUser = {
  userId: 1,
  username: "myUsername",
  role: "admin",
};
```

Since we need so many helper functions, we want to change `interface` to `models`. Then we have to update all use cases from

```ts
const myUser: LoginUser = { ... }
```

into

```ts
const myUser = new LoginUser(...)
```

This can be considered easy. But if we have a testing mock for this model like [this](https://github.com/bebit/usergram-front-v2/blob/8e50824a715667e7fc15ae8e97f3eb0f98f18e31/src/models/journey/UserJourneyPath.mock.ts), things can get complicated very quickly depending on how complicated the `interface` is. You may accidentally make the current tests fail, not to mention TypeScript complaining. It is doable, but it can generate unnecessary work. So if we make it a `class` from the first place, we can easily deal with it later.

### Add more getters to the class to get some values that require some calculations

```ts
// src/models/LoginUser.ts

export class LoginUser {
  constructor(
    public readonly userId: number, // <- must be camelCase
    public readonly username: string,
    public readonly role: string
  ) {}

  get isAdmin(): boolean {
    return this.role === "admin";
  }

  get isStaff(): boolean {
    return this.role === "staff";
  }
}
```

## What we should not do

### Creating unnecessary helper functions

We should not create util functions, or `computed` in Vue file when it is possible to use getters.

#### Example 1

**Bad**

```vue
// Home.vue

<template>
  <div>{{ isAdmin ? "admin" : "not admin" }}</div>
</template>

<script lang="ts" setup>
const user = new LoginUser(1, "myusername", "admin");

const isAdmin = computed(() => user.role === "admin");
</script>
```

```ts
// src/models/LoginUser.ts

export class LoginUser {
  constructor(
    public readonly userId: number, // <- must be camelCase
    public readonly username: string,
    public readonly role: string
  ) {}
}
```

**Good**

```vue
// Home.vue

<template>
  <div>{{ user.isAdmin ? "admin" : "not admin" }}</div>
</template>

<script lang="ts" setup>
const user = new LoginUser(1, "myusername", "admin");
</script>
```

```ts
// src/models/LoginUser.ts

export class LoginUser {
  constructor(
    public readonly userId: number, // <- must be camelCase
    public readonly username: string,
    public readonly role: string
  ) {}

  get isAdmin(): boolean {
    return this.role === "admin";
  }
}
```

#### Example 2

**Bad**

```vue
// Home.vue

<template>
  <div>{{ deviceAsString }}</div>
</template>

<script lang="ts" setup>
import { Device } from "@m/models/Device";
import { getTranslationOfDevice } from "@/utils/device";

const device = new Device();
const deviceAsString = computed(() =>
  getTranslationOfDevice(device.platformSubCategory)
);
</script>
```

```ts
// models/Device.ts

export class Device {
  constructor(public readonly platformSubCategory: number) {}
}
```

```ts
// utils/device.ts

export function getTranslationOfDevice(platformSubCategory: number): string {
  switch (platformSubCategory) {
    case PLATFORM_DEVICE_CATEGORY.IPHONE:
      return "iPhone";
    case PLATFORM_DEVICE_CATEGORY.ANDROID:
      return "Android";
    case PLATFORM_DEVICE_CATEGORY.IPAD:
      return "iPad";
  }

  return "";
}
```

**Good**

```vue
// Home.vue

<template>
  <div>{{ device.translatedName }}</div>
</template>

<script lang="ts" setup>
import { Device } from "@m/models/Device";

const device = new Device();
</script>
```

```ts
// models/Device.ts

export class Device {
  constructor(public readonly platformSubCategory: number) {}

  get translatedName(): string {
    switch (this.platformSubCategory) {
      case PLATFORM_DEVICE_CATEGORY.IPHONE:
        return "iPhone";
      case PLATFORM_DEVICE_CATEGORY.ANDROID:
        return "Android";
      case PLATFORM_DEVICE_CATEGORY.IPAD:
        return "iPad";
    }

    return "";
  }
}
```

### Dependencies injections from src/api

We should not inject dependencies from `src/api`, and models should not contain JSON parse methods (i.e. from API JSON to model). This should be done inside `src/api`.

**Bad**

```ts
// src/models/LoginUser.ts

export class LoginUser {
  constructor(
    public readonly userId: number, // <- must be camelCase
    public readonly username: string,
    public readonly role: string
  ) {}

  fromApiResponse(json): LoginUser {
    return new LoginUser(json.id, json.username, json.role);
  }
}
```

**Good**

```ts
// src/models/LoginUser.ts

export class LoginUser {
  constructor(
    public readonly userId: number, // <- must be camelCase
    public readonly username: string,
    public readonly role: string
  ) {}
}
```

```ts
// src/api/auth.ts

// If it's a simple model
export default class ApiLogin {
  static async login(payload: LoginPayload): Promise<LoginResponse> {
    const response = await http.post<LoginResponse>("/api/login", payload);
    return new LoginUser(response.user_id, response.username, response.role);
  }
}

// If it's a complicated model
export default class ApiLogin {
  static async login(payload: LoginPayload): Promise<LoginResponse> {
    const response = await http.post<LoginResponse>("/api/login", payload);
    return toLoginUserModel(response);
  }
}

function toLoginUserModel(json): LoginUser {
  return new LoginUser(json.id, json.username, json.role);
}
```
