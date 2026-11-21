# API Client, Server State & Forms Reference

## Generated Client

```json
{
  "scripts": {
    "api:generate": "openapi-typescript ../../docs/contracts/api/openapi.user-api.v1.json -o src/api/schema.gen.ts"
  }
}
```

```ts
// src/api/client.ts
import createClient from "openapi-fetch";
import type { paths } from "./schema.gen";

export const api = createClient<paths>({ baseUrl: config.apiUrl });

api.use({
  async onRequest({ request }) {
    const token = await auth.getAccessToken();      // from the OIDC user manager, in memory
    if (token) request.headers.set("Authorization", `Bearer ${token}`);
    return request;
  },
});
```

- Generate from the committed OpenAPI snapshot (the same one contract tests compare against); generated files are not edited by hand.
- A CI step regenerates and fails if the result differs from the committed file.
- The snapshot name identifies the configured Swagger document and API version (see `web-host-api`, swagger.md). Generate a separate client/schema for the admin document when the app uses it.

## Queries

```ts
export const paymentKeys = {
  all: ["payments"] as const,
  status: (orderId: string) => [...paymentKeys.all, "status", orderId] as const,
};

export function usePaymentStatus(orderId: string) {
  return useQuery({
    queryKey: paymentKeys.status(orderId),
    queryFn: async ({ signal }) => {
      const { data, error } = await api.GET("/api/v1/payments/{orderId}/status", {
        params: { path: { orderId } },
        signal,
      });
      if (error) throw toApiError(error);
      return data;
    },
  });
}
```

- Query keys come from one factory per feature; never inline arrays at call sites.
- Pass `signal` for cancellation. Set sensible `staleTime` per query instead of refetching everything on focus.
- Do not store query results in `useState`/global stores; derive from the query.

## Mutations

```ts
type CreatePaymentOperation = {
  body: CreatePaymentRequest; // Type derived from the generated OpenAPI schema
  idempotencyKey: string;
};

export function useCreatePayment() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async ({ body, idempotencyKey }: CreatePaymentOperation) => {
      const { data, error } = await api.POST("/api/v1/payments", {
        body,
        headers: { "Idempotency-Key": idempotencyKey },
      });
      if (error) throw toApiError(error);
      return data;
    },
    retry: false, // Explicit retries reuse the same operation variables
    onSuccess: () => queryClient.invalidateQueries({ queryKey: paymentKeys.all }),
  });
}
```

- Disable the submit button while `isPending`; send an `Idempotency-Key` for non-idempotent POSTs when the backend supports it.
- Create the operation object once, before its first `mutate` call: `{ body, idempotencyKey: crypto.randomUUID() }`. Retain that object in the form's operation state and pass it unchanged to manual or automatic retries. Never generate the key inside `mutationFn`. A timeout or lost response does not mean the server rolled back.
- Keep the key and request body together. Changing the payload starts a new operation with a new key. After an unknown outcome, resolve the previous operation's status before starting a replacement payment; do not silently create a new key on retry or remount.
- Test a committed payment whose response is lost: retrying the same operation returns its existing result and creates no second payment.
- Optimistic updates only where rollback is simple and the user sees the result immediately.

## Errors (ProblemDetails)
- `toApiError` turns RFC 9457 Problem Details responses into a typed `ApiError { status, type, title, errors? }`.
- Branch on `status` / `type`, never on `detail` text.
- 401 → trigger sign-in; 403 → "no access" page; 404 → not-found state; 409 → reload and show a conflict message; 5xx → generic error with retry.
- A global error boundary catches render errors; query errors are shown next to the affected component.

## Forms

```ts
const schema = z.object({
  orderId: z.string().min(1).max(100),
  amount: z.number().positive(),
});
type FormValues = z.infer<typeof schema>;

const form = useForm<FormValues>({ resolver: zodResolver(schema) });
```

- Client validation mirrors the backend validator for UX; the backend stays the authority.
- Map `ApiError.errors` (field → messages) to `form.setError(field, ...)`.
- Validation messages are i18n keys.
