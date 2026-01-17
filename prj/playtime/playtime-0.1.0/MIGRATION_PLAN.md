# Migration Plan: Hapi.js to Fastify

## Overview

This document outlines the step-by-step plan to create a new Fastify-based version of the Playtime project in a separate `playtime-fastify-0.1.0` folder. The original Hapi.js project will remain unchanged, allowing for side-by-side comparison and safe migration.

## Current Architecture Analysis

### Dependencies

- `@hapi/hapi`: Core Hapi.js framework
- `@hapi/vision`: View engine plugin for Handlebars
- `handlebars`: Template engine
- `uuid`: Utility library (no change needed)

### Current Structure

- **Server Setup** (`src/server.js`): Initializes Hapi server, registers Vision plugin, configures views
- **Routes** (`src/web-routes.js`): Route definitions using Hapi route configuration format
- **Controllers** (`src/controllers/`): Handler functions using Hapi's `(request, h)` signature
- **Views**: Handlebars templates (no changes needed)

## Migration Strategy

### Phase 0: Project Setup & Copy

#### 0.1 Create New Project Directory

Navigate to the parent directory and create the new Fastify project folder:

```bash
cd ..
mkdir playtime-fastify-0.1.0
cd playtime-fastify-0.1.0
```

#### 0.2 Copy Project Files

Copy the entire project structure from the original Hapi.js project:

```bash
# From the parent directory (playtime/)
cp -r playtime-0.1.0/* playtime-fastify-0.1.0/
# Or if using rsync:
rsync -av --exclude='node_modules' --exclude='.git' playtime-0.1.0/ playtime-fastify-0.1.0/
```

**Files to copy:**

- `src/` directory (all files)
- `package.json`
- `.gitignore` (if present)
- `CHANGELOG.md` (optional, for reference)

**Files to exclude:**

- `node_modules/` (will be recreated)
- `.git/` (or initialize new git repo)

#### 0.3 Initialize New Project

```bash
cd playtime-fastify-0.1.0
# Initialize new git repository (optional)
git init
git add .
git commit -m "Initial Fastify migration - copied from Hapi.js project"
```

### Phase 1: Dependencies Update

#### 1.1 Remove Hapi Dependencies

In the `playtime-fastify-0.1.0` directory:

```bash
npm uninstall @hapi/hapi @hapi/vision
```

#### 1.2 Install Fastify Dependencies

```bash
npm install fastify @fastify/view @fastify/formbody handlebars
```

**Note**: The `uuid` package should already be in package.json and can remain as-is.

**Note**:

- `@fastify/view` is the Fastify organization's scoped package for the point-of-view plugin. It provides view engine support for Fastify, including Handlebars.
- `@fastify/formbody` is needed to parse URL-encoded form data from POST requests (signup, login, addplaylist routes).

### Phase 2: Server Setup Migration (`playtime-fastify-0.1.0/src/server.js`)

#### 2.1 Import Changes

- Replace: `import Hapi from "@hapi/hapi";`
- With: `import Fastify from "fastify";`

- Replace: `import Vision from "@hapi/vision";`
- With:
  ```javascript
  import formBody from "@fastify/formbody";
  import pointOfView from "@fastify/view";
  ```
- Replace: `import Handlebars from "handlebars";`
- With: `import Handlebars from "handlebars";` (no change)

#### 2.2 Server Initialization

- Replace: `Hapi.server({ port: ... })`
- With: `Fastify({ logger: true })` (or `{ logger: false }` if preferred)

#### 2.3 Plugin Registration

- Replace: `await server.register(Vision);`
- With:
  ```javascript
  await fastify.register(formBody);
  await fastify.register(pointOfView, { engine: { handlebars: Handlebars } });
  ```

**Note**: Register `@fastify/formbody` before the view plugin to ensure form data parsing works for POST routes.

#### 2.4 View Configuration

- Replace Hapi's `server.views()` configuration
- With Fastify's view configuration in the `pointOfView` register call:
  ```javascript
  await fastify.register(pointOfView, {
    engine: { handlebars: Handlebars },
    root: path.join(__dirname, "views"),
    layout: "./layouts/layout.hbs",
    viewExt: "hbs",
    options: {
      partials: {
        header: "./partials/header.hbs",
        footer: "./partials/footer.hbs",
      },
    },
  });
  ```

#### 2.5 Server Start

- Replace: `await server.start();`
- With: `await fastify.listen({ port: process.env.PORT || 3000 });`

- Replace: `server.info.uri`
- With: `fastify.server.address()` or construct the URL manually

### Phase 3: Route Configuration Migration (`playtime-fastify-0.1.0/src/web-routes.js`)

#### 3.1 Route Format Changes

- Hapi uses: `{ method: "GET", path: "/", config: { handler: ... } }`
- Fastify uses: `{ method: "GET", url: "/", handler: ... }`

**Key Changes:**

- `path` → `url`
- `config.handler` → `handler` (flatten structure)

#### 3.2 Update Route Definitions

Convert all routes from:

```javascript
{ method: "GET", path: "/", config: accountsController.index }
```

To:

```javascript
{ method: "GET", url: "/", handler: accountsController.index }
```

### Phase 4: Controller Migration (`playtime-fastify-0.1.0/src/controllers/`)

#### 4.1 Handler Function Signature

- **Hapi**: `function (request, h) { ... }`
- **Fastify**: `async function (request, reply) { ... }`

#### 4.2 Response Methods Migration

| Hapi Method                | Fastify Equivalent             |
| -------------------------- | ------------------------------ |
| `h.view("template", data)` | `reply.view("template", data)` |
| `h.redirect("/path")`      | `reply.redirect("/path")`      |
| `h.response(data)`         | `reply.send(data)`             |

#### 4.3 Request Object Access

- **Payload**: `request.payload` → `request.body` (for POST/PUT requests)
- **Query**: `request.query` → `request.query` (same)
- **Params**: `request.params` → `request.params` (same)

#### 4.4 Specific Controller Updates

**accounts-controller.js:**

- Replace all `h.view()` with `reply.view()`
- Replace all `h.redirect()` with `reply.redirect()`
- Replace `request.payload` with `request.body`

**dashboard-controller.js:**

- Replace `h.view()` with `reply.view()`
- Replace `h.redirect()` with `reply.redirect()`
- Replace `request.payload` with `request.body`

### Phase 5: View Configuration Details

#### 5.1 Layout Support

Fastify's `point-of-view` plugin supports layouts but the configuration is different. Need to verify:

- Layout path configuration
- Partial paths configuration
- Template caching options

#### 5.2 Layout Path Resolution

- Ensure `layoutPath` is correctly mapped
- Verify `partialsPath` configuration
- Test layout rendering

### Phase 6: Testing & Validation

#### 6.1 Functional Testing Checklist

- [ ] Server starts successfully
- [ ] All routes respond correctly
- [ ] Views render with correct layouts
- [ ] Partials are included properly
- [ ] Redirects work correctly
- [ ] POST requests handle payload correctly
- [ ] Static assets (if any) are served correctly

#### 6.2 Edge Cases

- [ ] Error handling
- [ ] 404 responses
- [ ] Invalid form submissions
- [ ] Database operations

## Detailed File-by-File Changes

All changes will be made in the `playtime-fastify-0.1.0/` directory. The original `playtime-0.1.0/` directory remains unchanged.

### `playtime-fastify-0.1.0/package.json`

1. Remove `@hapi/hapi` and `@hapi/vision` dependencies
2. Add `fastify`, `@fastify/view`, and `@fastify/formbody` dependencies
3. Keep `handlebars` and `uuid` (already present)
4. Update project name if desired (e.g., `"name": "playtime-fastify-0.1.0"`)

### `playtime-fastify-0.1.0/src/server.js`

1. Update imports (replace Hapi imports with Fastify imports)
2. Replace server initialization (`Hapi.server()` → `Fastify()`)
3. Update plugin registration (replace Vision with formBody and pointOfView)
4. Update view configuration (convert Hapi `server.views()` to Fastify `pointOfView` config)
5. Update server start logic (`server.start()` → `fastify.listen()`)

### `playtime-fastify-0.1.0/src/web-routes.js`

1. Change `path` property to `url` in all route definitions
2. Flatten route structure (move `handler` from `config` object to root level of route definition)
3. Update all 7 route definitions accordingly

### `playtime-fastify-0.1.0/src/controllers/accounts-controller.js`

1. Change handler signature: `(request, h)` → `(request, reply)` in all handlers
2. Replace all `h.view()` calls with `reply.view()`
3. Replace all `h.redirect()` calls with `reply.redirect()`
4. Replace `request.payload` with `request.body` in POST handlers

### `playtime-fastify-0.1.0/src/controllers/dashboard-controller.js`

1. Change handler signature: `(request, h)` → `(request, reply)` in all handlers
2. Replace `h.view()` with `reply.view()`
3. Replace `h.redirect()` with `reply.redirect()`
4. Replace `request.payload` with `request.body` in POST handlers

### `playtime-fastify-0.1.0/src/models/db.js`

- **No changes required** (framework-agnostic)

### View Files (`playtime-fastify-0.1.0/src/views/`)

- **No changes required** (Handlebars templates are framework-agnostic)
- All view files are copied as-is from the original project

## Migration Steps Execution Order

1. **Setup**: Create `playtime-fastify-0.1.0` directory in parent folder
2. **Copy**: Copy all project files from `playtime-0.1.0/` to `playtime-fastify-0.1.0/` (excluding `node_modules`)
3. **Dependencies**: Update `package.json` in `playtime-fastify-0.1.0/` directory
   - Remove `@hapi/hapi` and `@hapi/vision`
   - Add `fastify`, `@fastify/view`, and `@fastify/formbody`
4. **Install**: Run `npm install` in `playtime-fastify-0.1.0/` directory
5. **Server**: Migrate `playtime-fastify-0.1.0/src/server.js`
6. **Routes**: Migrate `playtime-fastify-0.1.0/src/web-routes.js`
7. **Controllers**: Migrate both controller files in `playtime-fastify-0.1.0/src/controllers/`
8. **Test**: Start server from `playtime-fastify-0.1.0/` directory and test each route manually
9. **Fix**: Address any issues found during testing
10. **Verify**: Ensure all functionality works as expected and matches original Hapi.js behavior

## Potential Challenges & Solutions

### Challenge 1: View Layout Configuration

**Issue**: Fastify's `point-of-view` may have different layout handling than Hapi Vision.

**Solution**:

- Review `@fastify/view` documentation for layout configuration
- May need to adjust layout path or use a different approach
- Consider using `fastify-view-engine` if layouts are complex

### Challenge 2: Form Data Parsing

**Issue**: Fastify may handle form data differently. The current code uses POST requests with form data (signup, login, addplaylist).

**Solution**:

- Install `@fastify/formbody` plugin for handling `application/x-www-form-urlencoded` form data
- Register it in `server.js`: `await fastify.register(require('@fastify/formbody'))`
- Form data will then be accessible via `request.body` (same as JSON payloads)
- Alternative: Convert forms to send JSON and use Fastify's built-in JSON parser

### Challenge 3: Route Registration

**Issue**: Fastify route registration syntax is slightly different.

**Solution**:

- Use `fastify.route()` or register routes directly
- Can use `fastify.register()` for route modules if desired

### Challenge 4: Async Handler Support

**Issue**: Fastify handlers are async by default.

**Solution**:

- Ensure all handlers are marked as `async` or return promises
- Fastify automatically handles promises

## Performance Considerations

### Advantages of Fastify

- Generally faster than Hapi (lower overhead)
- Built-in JSON schema validation support
- Modern async/await support
- Active development community

### Things to Watch

- View rendering performance (should be similar)
- Plugin ecosystem differences
- Learning curve for Fastify-specific features

## Rollback Plan

Since we're creating a separate project, rollback is straightforward:

1. **Original project safe**: The original `playtime-0.1.0/` directory remains completely unchanged
2. **Delete and restart**: If needed, simply delete `playtime-fastify-0.1.0/` directory and start over
3. **Keep copy**: You can keep multiple attempts by renaming the folder (e.g., `playtime-fastify-0.1.0-v2`)
4. **Document issues**: Note any problems encountered for future reference

## Project Structure After Migration

```
playtime/
├── playtime-0.1.0/              # Original Hapi.js project (unchanged)
│   ├── src/
│   ├── package.json
│   └── ...
└── playtime-fastify-0.1.0/      # New Fastify project
    ├── src/
    │   ├── server.js            # (migrated)
    │   ├── web-routes.js        # (migrated)
    │   ├── controllers/         # (migrated)
    │   ├── models/              # (copied as-is)
    │   └── views/               # (copied as-is)
    ├── package.json             # (updated dependencies)
    └── node_modules/            # (new install)
```

## Post-Migration Optimization Opportunities

1. **Schema Validation**: Fastify has excellent JSON schema validation - consider adding validation schemas
2. **Logging**: Fastify has built-in Pino logger - consider utilizing it
3. **Error Handling**: Fastify has structured error handling - consider implementing proper error handlers
4. **Rate Limiting**: Consider adding `@fastify/rate-limit` plugin
5. **Security**: Consider adding `@fastify/helmet` for security headers

## Estimated Effort

- **Project Setup & Copy**: 5-10 minutes
- **Dependencies Update**: 5 minutes
- **Server Migration**: 15-20 minutes
- **Routes Migration**: 10 minutes
- **Controllers Migration**: 15-20 minutes
- **Testing & Debugging**: 20-30 minutes
- **Total**: ~1.5-2 hours

## References

- [Fastify Documentation](https://www.fastify.io/)
- [@fastify/view Documentation](https://github.com/fastify/point-of-view)
- [Fastify Route Documentation](https://www.fastify.io/docs/latest/Reference/Routes/)
