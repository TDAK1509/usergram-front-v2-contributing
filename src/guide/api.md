# src/api

This directory containing files that are responsible for making API calls, and return data as models. This directory and [src/models](models.html) work together.

Let's say you want to create a new file to handle the API `/api/login` like this:

| api          | method | payload                                  | response                                              |
| ------------ | ------ | ---------------------------------------- | ----------------------------------------------------- |
| `/api/login` | POST   | `{ username: string, password: string }` | `{ user_id: number, username: string, role: string }` |

Here is what you need to do:

- Create a new file `src/api/login.ts`
- Create types for raw API responses for this API.

```ts
// src/api/login.ts
interface LoginPayload {
  username: string;
  password: string;
}

interface LoginResponse {
  user_id: number;
  username: string;
  role: string;
}
```

- Create a new file to handle model for login user `src/models/LoginUser.ts`

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

:exclamation: **Please use `class` instead of `interface` for model, it will give more control.**

- Create a class to handle API calls

```ts
// src/api/login.ts
import HttpClient from "./HttpClient";
import { LoginUser } from "@/models/LoginUser";

interface LoginPayload {
  username: string;
  password: string;
}

interface LoginResponse {
  user_id: number;
  username: string;
  role: string;
}

const http = new HttpClient();

export default class ApiLogin {
  static async login(payload: LoginPayload): Promise<LoginResponse> {
    const response = await http.post<LoginResponse>("/api/login", payload);
    return new LoginUser(response.user_id, response.username, response.role);
  }
}
```

:exclamation: **Converting from API response JSON to models should be done in this directory instead of inside [src/models](models.html). See [Don't inject dependencies from src/api](models.html#don-t-inject-dependencies-from-src-api).**
