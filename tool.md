### Approach 1 (NestJS + React Vite + Client side router)

With NestJS, I’d assume we are going to use it as a backend server with endpoints that responds with JSON format right like an API?

A controller which looks like this

```ts
// src/carriers/carriers.controller.ts

import { Controller, Get } from "@nestjs/common";
import { CarriersService } from "./carriers.service";
import { Carrier } from "./carrier.model";

@Controller("api/carriers")
export class CarriersController {
  constructor(private readonly carriersService: CarriersService) {}

  @Get()
  async getCarriers(): Promise<Carrier[]> {
    return this.carriersService.getCarriers();
  }
}
```

```ts
// carrier.model.ts
export type Carrier = {
  id: number;
  name: string;
};
```

To serve a UI, we’ll use React for frontend. It’ll be React with ViteJS. Eventually we’ll be needing to serve different pages like (example):

- /login - for login page
- /register - for registration
- /quotations - for customer to view their list of quotations
- /quotations/new - for customer to inquire pricing
  And more…

Which we will be reaching out to libraries that makes it easy to organize client side routes, code splitting by pages, lazy loading, & browser history manipulation. Tools like:

- React Router
- TanStack Router
- And others

I guess the folder structure would look like

```
backend (nestjs)
  package.json
frontend (react vite)
  package.json
```

Running the app in development would be running `npm run dev` in two terminals (one for backend, and one for frontend)

A frontend page/component say the lists of carriers page would have something like this:

```tsx
// frontend/src/pages/ListCarriers.tsx

export default function ListCarriers() {
  const [carriers, setCarriers] = React.useState([]);

  React.useEffect(() => {
    async function listCarriers() {
      const nestJsCarrierEndpoint = "http://localhost:3000/api/carriers";
      const response = await fetch(nestJsCarrierEndpoint);
      setCarriers(await response.json());
    }
    listCarriers();
  }, []);


  return (
    <ul>
      {carriers.map(c => (<li key={c.id}>{c.name}</li>))}
    </li>
  )
}
```

Would be nice if the type from `carrier.model.ts` which is in the backend folder can be imported in the frontend. Something like:

```tsx
import { Carrier } from "backend/src/carriers/carrier.model";

const [quotations, setQuotations] = React.useState<Carrier[]>([]);
```

But I think for that to work, we might need [Workspaces](https://docs.npmjs.com/cli/v10/using-npm/workspaces) or Monorepo tooling like [NX](https://nx.dev/) or [Turborepo](https://turbo.build/). Manual approach is to create another `carrier.model` file in the frontend directory but this would lead to duplicate types (one for FE and one for BE).

### Approach 2 (NestJS + NextJS)

Assume we have the same NestJS controller for the `/api/carriers` endpoint like Approach #1 and a similar folder stucture as well:

```
backend (nestjs)
  package.json
frontend (nextjs)
  package.json
```

Since NextJs comes with built it routing (file based routing). No need for client side router libraries.

```tsx
// frontend/src/app/carriers/page.tsx

export default async function ListCarriers() {
  const nestJsCarrierEndpoint = "http://localhost:3000/api/carriers";
  const response = await fetch(nestJsCarrierEndpoint);
  const carriers = await response.json()

  return (
    <ul>
      {carriers.map(c => (<li key={c.id}>{c.name}</li>))}
    </li>
  )
}

```

NextJS app router does [React Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components) which can perform data fetching from the server. In the above approach it is server-side rendered. Unlike Approach #1 it doesn't need `useEffect` for fetching carriers in the client side which doesn't suffer from [requests waterfalls](https://blog.sentry.io/fetch-waterfall-in-react/).

Notice that `carriers` would have a type of `any`, which it still kind-a similar pattern to Approach #1.

### Approach #3 (NextJS for backend/frontend)

[NextJS](https://nextjs.org/) is a fullstack framework which is running in the server then can prerender React components in server and then hydrate in client.

```tsx
// src/app/carriers/page.tsx

import carriersService from "./carriers.service";

export default async function ListCarriers() {
  const carriers = await carriersService.getCarriers(); // assume that getCarriers returns an array with Carrier type {id: number; name: string}

  return (
    <ul>
      {carriers.map(c => (<li key={c.id}>{c.name}</li>))}
    </li>
  )
}

```

With Typescript [type inference](https://www.typescriptlang.org/docs/handbook/type-inference.html) the `carriers` variable will have a `Carrier[]` type. No need to explicitly type it as `Carrier[]`.

#### Making API endpoints in NextJS

##### For GET /api/carriers

```tsx
// src/app/api/carriers/route.ts

import carriersService from "./carriers.service";

export async function GET() {
  return carriersService.getCarriers(); // returns a Carrier[]
}
```

##### For GET /api/carriers/:carrierId

```tsx
// src/app/api/carriers/[carrierId]/route.ts

import carriersService from "./carriers.service";

export async function GET(
  request: Request,
  { params }: { params: { carrierId: string } }
) {
  const carrierId = Number(params.carrierId);
  return carriersService.getCarrierById(carrierId); // returns a Carrier | null.
}
```
