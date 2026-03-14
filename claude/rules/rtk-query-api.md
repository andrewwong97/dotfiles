# RTK Query API Rules

## Pattern

When adding a new backend-facing API domain (e.g. admin, billing, notifications):

1. **Create a new RTK Query API slice** in `frontend/src/api/<domain>Api.ts`.
   - Import the shared `axiosBaseQuery` from `@/api/baseQuery`.
   - Define a unique `reducerPath` (e.g. `'adminApi'`).
   - Define `tagTypes` for cache invalidation.
   - Export auto-generated hooks (`useGet...Query`, `use...Mutation`).

2. **Register the slice** in `frontend/src/store.ts`:
   - Add the reducer: `[newApi.reducerPath]: newApi.reducer`
   - Add the middleware: `.concat(newApi.middleware)`

3. **Use hooks in components** — do NOT use raw `api.get()`/`api.post()` calls.
   - Queries handle loading, error, and caching automatically.
   - Mutations return `.unwrap()` for try/catch in handlers.
   - Use `skip` option to gate queries on auth state.

4. **Stories** use prepopulated RTK Query cache (use the `buildMockStore` pattern with `preloadedState`).

## Reference files

- Shared base query: `frontend/src/api/baseQuery.ts`
- Store registration: `frontend/src/store.ts`
- API slices: `frontend/src/api/<domain>Api.ts`
