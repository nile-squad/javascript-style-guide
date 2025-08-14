# JavaScript / TypeScript Style Guide 

> => roughly 9974 tokens, 39898 characters and 6197 words - summarizing contributions are welcome!

**Version:** 1.0  
**Date:** August 14, 2025  
**Author:** Hussein Kizz

**Philosophy:** Good code is for machines, great code is written for humans first and machines. Code should be like a poem: fun to read, vivid, illustrative, self-sustaining, short and wide where needed, structured, full of humor with rhythm and flow, interesting and makes the reader want to meet the author, and each line leading to more tense to find out the whole story. No line is a waste in poetry!

Code Practices is an evolution, so how we code changes as our skills and beliefs change over time, but for as date of this version, this is how we write code at Nile Squad Labz.

## 1. Non-Negotiable Rules

These aren't suggestions - they're the foundation of everything we build. Think of them as the grammar rules of our code language.

- ❌ **No classes** - Use functions, factories, closures
- ✅ **ES6+ required** - `const`, `let`, arrow functions, destructuring, modules
- ✅ **Max 400 LOC per file** - Split before exceeding (maintainability limit)
- ✅ **Task-driven workflow** - Create `task.md` before non-trivial changes
- ✅ **Functional composition** - Build complex behavior from simple functions

### 1.1 Why These Rules Matter

- **No Classes:** Classes hide state and create `this` confusion. Functions are predictable - same input, same output. They compose naturally and test easily.
- **ES6+:** Modern syntax prevents entire categories of bugs. `const` prevents accidental reassignment, destructuring makes data flow obvious, arrow functions eliminate `this` surprises.
- **Lines Of Code Limits:** Your brain can only hold so much context. After 400 lines, you're not reading code anymore - you're getting lost in it.
- **Task Documentation:** Writing down your plan before coding prevents scope creep and creates an audit trail. Future you will thank present you.

## 2. Core Programming Principles

These principles translate beautifully when you think in terms of functions instead of classes.

### 2.1 Single Responsibility Principle

Each function should have one reason to change. If you can't describe what your function does in one sentence without using "and," it's doing too much.

**The Test:** Can you name your function with a verb + noun that clearly expresses its purpose? `validateEmail`, `formatCurrency`, `calculateTax` - these names tell you exactly what the function does and nothing else.

**❌ Wrong Way (multiple responsibilities):**

```ts
function processUserData(userData) {
  // Validation responsibility
  if (!userData.email || !userData.name) {
    throw new Error('Invalid data');
  }
  
  // Formatting responsibility  
  const formatted = {
    email: userData.email.toLowerCase(),
    name: userData.name.trim()
  };
  
  // Database responsibility
  database.saveUser(formatted);
  
  // Notification responsibility
  emailService.sendWelcome(formatted.email);
  
  return formatted;
}
```

**✅ Right Way (single responsibilities):**

```ts
const validateUserData = (userData) => {
  if (!userData.email || !userData.name) {
    return { status: false, message: 'Email and name are required' };
  }
  return { status: true };
};

const formatUserData = (userData) => ({
  email: userData.email.toLowerCase(),
  name: userData.name.trim()
});

const saveUser = async (database, userData) => {
  const { err, result } = await safeTry(() => database.saveUser(userData));
  if (err) return { status: false, message: 'Failed to save user' };
  return { status: true, data: result };
};

const sendWelcomeEmail = async (emailService, email) => {
  const { err } = await safeTry(() => emailService.sendWelcome(email));
  if (err) return { status: false, message: 'Failed to send welcome email' };
  return { status: true };
};

// Compose them together
const processUser = async (userData, { database, emailService }) => {
  const validation = validateUserData(userData);
  if (!validation.status) return validation;
  
  const formatted = formatUserData(userData);
  
  const saveResult = await saveUser(database, formatted);
  if (!saveResult.status) return saveResult;
  
  await sendWelcomeEmail(emailService, formatted.email);
  
  return { status: true, data: formatted };
};
```

> Caution: Sometimes it makes sense for a function or module to be long and verbose, especially if pieces are too intricate and logic is not reasonable when fragmented, in such cases it's fine to do more than a thing, or usually day to day 2 to 3 things won't hurt.

### 2.2 Dependency Injection

Depend on abstractions (injected functions) not concrete implementations. This is the secret sauce that makes your code testable and flexible.

**❌ Wrong Way (hard dependencies):**

```ts
import { database } from '../database/connection';
import { emailService } from '../services/email';
import { logger } from '../utils/logger';

function createUser(userData) {
  logger.info('Creating user');
  const user = database.users.create(userData);
  emailService.sendWelcome(user.email);
  return user;
}
```

**✅ Right Way (injected dependencies):**

```ts
// Dependencies are explicit parameters
const createUserService = ({ database, emailService, logger }) => {
  const createUser = async (userData) => {
    logger.info('Creating user');
    
    const { err, result } = await safeTry(() => database.users.create(userData));
    if (err) {
      logger.error('Failed to create user', err);
      return { status: false, message: 'User creation failed' };
    }
    
    // Non-blocking email (fire and forget)
    emailService.sendWelcome(result.email).catch(emailErr => 
      logger.warn('Welcome email failed', emailErr)
    );
    
    return { status: true, data: result };
  };
  
  return { createUser };
};

// Usage - dependencies are clear and testable
const userService = createUserService({ 
  database: realDatabase,
  emailService: realEmailService,
  logger: realLogger
});

// Testing becomes trivial
const testUserService = createUserService({
  database: mockDatabase,
  emailService: mockEmailService, 
  logger: mockLogger
});
```

**Why DI Works:**

- Makes dependencies explicit and visible
- Enables easy testing with mocks
- Allows swapping implementations without code changes
- Functions become pure and predictable
- Eliminates hidden imports buried in functions

> Caution: for things like loggers, and singleton instances you may actually just want to import and just use freely, don't do DI for utilities and cases it doesn't work well, consitency of approach is more preferred sometimes than being complaint.

### 2.3 Open/Closed Principle  

Extend behavior through composition, not modification. Instead of adding more `if/else` statements to existing functions, create new functions and compose them together.

**Why This Matters:** When you modify working code to add features, you risk breaking what already works. When you compose new behavior from existing pieces, the old stuff keeps working while new stuff gets added safely.

### 2.4 Interface Segregation

Expose small, focused APIs. Better to have five small services than one kitchen-sink service that does everything.

**The Human Factor:** Small interfaces are easier to understand, document, and test. When someone needs user validation, they don't want to import an entire user management system.

### 2.5 Additional Principles

**KISS:** Choose the simplest solution that works.

**YAGNI:** Don't build features until they're actually needed. Every line of code is a liability.

**DRY:** Extract common patterns into reusable utilities, but don't over-abstract small, domain-specific code. Three instances might be coincidence. Five instances are a pattern worth abstracting.

## 3. Core Patterns

These patterns solve real problems we encounter every day. Master these, and you'll handle 90% of coding situations elegantly.

### Factory Pattern (Standard)

Factories group related functions and make dependencies explicit. They're like constructors for functions - they set up everything a group of related operations needs to work.

```ts
// ✅ Factory groups related operations with clear dependencies
  const getUser = async (id) => {
    const { err, result } = await safeTry(() => database.findUser(id));
    if (err) return { status: false, message: 'User not found' };
    return { status: true, data: result };
  };
  
  const createUser = async (userData) => {
    const { err } = safeTry(() => validator.validateUser(userData));
    if (err) return { status: false, message: 'Invalid user data' };
    
    const { err: dbErr, result } = await safeTry(() => database.insertUser(userData));
    if (dbErr) return { status: false, message: 'Failed to create user' };
    return { status: true, data: result };
  };
  
  return { getUser, createUser };
};

// Usage - dependencies are clear and testable
const userService = createUserService(database, validator);
```

**Why Factories Work:** They make dependencies explicit (no hidden imports), enable easy testing (inject mocks), and group related functionality naturally.

### Additional Functional Programming Guidelines

- Don't be a functional purist - be flexible to break the rules when it makes sense. Functions may not be pure all the time.
- Always ask: can this be broken down further?
- Think in composition and data flow, pipelines, procedures - not classes
- Functions can return objects, objects can expose functions as methods or their properties (see factory pattern)
- Use design patterns, but in a way that makes sense for how they will be consumed
- Think API first - how will this be consumed? That determines the implementation

### Error Handling Philosophy

Here's the thing about errors: they're not exceptional - they're part of normal program flow. Network calls fail, users enter bad data, databases go down. Plan for it.

**Ideal Approach:**

- **Fail Fast:** Throw for developer errors (missing config, wrong types)
- **Fail Safe:** Handle gracefully for runtime errors (network, user input)
- **Explicit States:** Use Result patterns instead of throwing in business logic
- **Context:** Always provide enough information to understand what went wrong but don't expose inner workings in error messages. Avoid technical language and jargon in error messages, and never show code. Use error codes that can be traced in monitoring systems.
- **Validation** Never trust user data, always validate it, sanitize and expect missing fields or wrong data types, use validation libraries to enforce schema on all layers.
- **Plan For Errors** Plan for failure and nothing fails, if something can go wrong, it will make your code aware of the fact in adavance, safe guard the scenarios day one.

Make error handling consistent, log errors, don't play try and catch everywhere unless needed, instead make a utility wrapper around try catch like safe try, this can be used instead to keep your code cleaner.

```ts
// ✅ Standard Pattern - Consistent error handling
const { err, result } = await safeTry(() => api.fetchUser(id));
if (err) {
  // Log with context but don't expose internal details to users
  return { status: false, message: 'User not found' };
}

// ✅ For both sync and async operations
const { err: validationErr } = safeTry(() => validateEmail(email));
if (validationErr) {
  return { status: false, message: 'Invalid email format' };
}
```

### Early Returns (Guard Clauses) And Defensive Programming

Stop nesting your happy path inside layers of conditions. Validate inputs first, handle edge cases early, then let your main logic flow naturally. If something can fail, it will, so anticipate it and plan for failure so that nothing fails eventually.

```ts
// ✅ Right Way - Happy path is obvious, no nesting
function processUser(user, permissions) {
  // Guard clauses - fail fast on invalid inputs
  if (!user) return { status: false, message: 'User object is required' };
  if (!user.id) return { status: false, message: 'User must have an ID' };
  if (!user.active) return { status: false, message: 'User account is inactive' };
  if (!permissions?.length) return { status: false, message: 'User has no permissions' };
  
  // Business logic validation
  if (!permissions.includes('READ')) {
    return { status: false, message: 'User lacks read permissions' };
  }
  
  // Happy path - clear and unindented
  const processedUser = {
    ...user,
    lastProcessed: Date.now(),
    permissions
  };
  
  return { status: true, data: processedUser };
}
```

You can always refactor checking logic into something like checkUserRequirements to reduce ambiquity when guards are too much too many.

**Why This Works:** Your brain reads code linearly. When the happy path is buried inside nested conditions, you lose track of what the function actually does when everything goes right. Besides, code is evaluated top to bottom, save some compute and enable optimizations at runtime by handling bad cases early on.

### Algebraic Data Types (Explicit State)

Make impossible states impossible by using types that can only represent valid combinations of data. It's like a social contract, involved parties always know what to expect, no suprises, unless impossible, return same shape everywhere.

```ts
// ✅ ADT Pattern - Status is boolean, data structure is explicit
type Result<T> = 
  | { status: true; data: T }
  | { status: false; message: string };

// Usage - TypeScript enforces handling all cases
function handleUserResult(result: Result<User>) {
  if (result.status) {
    // TypeScript knows result.data exists here
    console.log(`Welcome ${result.data.name}`);
  } else {
    // TypeScript knows result.message exists here
    console.error(`Error: ${result.message}`);
  }
}
```

**The Power:** You can't accidentally access `result.data` when status is false, and you can't forget to handle the error case.

## 4. Code Structure & Organization

Good organization isn't about following rules - it's about helping future developers (including yourself) find what they need quickly.

### File Organization Philosophy

- **Max 400 lines per file** - When you hit this limit, split the file. Your future self will thank you unless unity and verbosity is needed, composition is always better at scale.
- **Use barrel files (`index.ts`)** for module public APIs - They create clean boundaries between modules, grouped domain related modules into own directory, let index file export them, import in other modules via namespace or explicitly if only a few methods are used.
- **Group by feature/domain**, not just technical layer - Related things should live near each other when possible, but more generalized organization the better.
- **Prefer namespace imports** - `import * as users from './users'` makes the source of functions obvious

### Why Domain Organization Matters

When you group by technical layer (all services in one place, all utilities in another), you scatter related functionality across your codebase. When you group by business domain, everything related to users lives together, everything related to orders lives together. Finding and changing related functionality becomes natural.

**Balance:** While domain organization is preferred for business logic, truly generic utilities (like safeTry, formatters, validators) should live in a shared utils directory to avoid duplication across domains.

### Naming Conventions (Self-Documenting Code)

Names should tell you what something is or does without requiring you to read the implementation.

- **Functions:** `verbNoun` (`getUser`, `createOrder`, `validateEmail`)
- **Booleans:** `is/has/can/should` (`isActive`, `hasPermission`, `canDelete`, `shouldRetry`)
- **Constants:** `UPPER_CASE_SNAKE` (`MAX_RETRY_ATTEMPTS`, `DEFAULT_TIMEOUT`)
- **Files:** `kebab-case` (`user-service.ts`, `order-validation.ts`)
- **Types:** `PascalCase` (`UserProfile`, `OrderStatus`, `ApiResponse`)

### Object Parameters (Self-Documenting APIs)

Stop making developers remember the order of parameters. Use object parameters for any function that takes more than two arguments.

```ts
Wrong - Hard to remember, easy to mix up
function createUser(name, email, isAdmin, createdAt, preferences) {}

Right - Self-documenting, optional parameters, extensible
interface CreateUserParams {
  name: string;
  email: string;
  isAdmin?: boolean;
  createdAt?: number;
  preferences?: UserPreferences;
}

function createUser({ 
  name, 
  email, 
  isAdmin = false, 
  createdAt = Date.now(),
  preferences = {}
}: CreateUserParams) {
  return {
    id: generateId(),
    name: name.trim(),
    email: email.toLowerCase(),
    isAdmin,
    createdAt,
    preferences
  };
}

// Usage is clear and flexible
createUser({ 
  name: 'Alice Johnson', 
  email: 'alice@example.com',
  preferences: { theme: 'dark', notifications: true }
});
```

## 5. Async Programming Patterns

Async code is where bugs love to hide. These patterns help you write predictable, debuggable async operations.

### The safeTry Pattern

This is your best friend for handling both sync and async operations that might fail. It turns exceptions into explicit return values you can check.

```ts
// For async operations
const { err, result } = await safeTry(() => api.fetchUser(id));
if (err) {
  // Handle error explicitly - no surprises
  return { status: false, message: 'Failed to fetch user' };
}

// For sync operations that might throw
const { err: parseErr, result: parsed } = safeTry(() => JSON.parse(jsonString));
if (parseErr) {
  return { status: false, message: 'Invalid JSON format' };
}
```

### Sequential Operations

It's easy to debug, you can just add break points or logs on each step, procedure by procedure, than any other fuzzy way that's probably harder to follow on and comprehend.

**Sequential (one depends on another):** Use await and handle each step explicitly

```ts
async function processOrder(orderData) {
  // Step 1: Validate input
  const { err: validationErr } = safeTry(() => validateOrderData(orderData));
  if (validationErr) {
    return { status: false, message: 'Invalid order data' };
  }
  
  // Step 2: Get user (depends on validation passing)
  const { err: userErr, result: user } = await safeTry(() => 
    userService.getUser(orderData.userId)
  );
  if (userErr) {
    return { status: false, message: 'User not found' };
  }
  
  // Step 3: Create order (depends on user existing)
  const { err: orderErr, result: order } = await safeTry(() => 
    orderService.create({ ...orderData, user })
  );
  if (orderErr) {
    return { status: false, message: 'Failed to create order' };
  }
  
  return { status: true, data: order };
}
```

### When to Use .then() vs await

This is one of those choices that seems small but shapes how readable your async code becomes. Both work, but they excel in different situations.

**Use `await` for sequential logic** where you need explicit error handling and clear dependencies between operations. The safeTry pattern works beautifully with await because you get explicit error values you can check.

**Use `.then()` for simple transforms and functional chains** where you're just massaging data through a pipeline. It keeps transformation code concise and expressive.

**The Golden Rule:** Never mix `await` and `.then()` in the same function unless you have a compelling reason and document it.

#### Examples

**❌ Wrong Way (lost error context):**

```ts
async function run() {
  try {
    const a = await fnA();
    const b = await fnB(a);
    const c = await fnC(b);
  } catch (e) {
    // Which operation failed? We lost context
    console.error('Something went wrong:', e);
  }
}
```

**✅ Right Way (await + safeTry for sequential logic):**

```ts
async function run() {
  const { err: errA, result: a } = await safeTry(() => fnA());
  if (errA) return handleError('fnA failed', errA);
  
  const { err: errB, result: b } = await safeTry(() => fnB(a));
  if (errB) return handleError('fnB failed', errB);
  
  const { err: errC, result: c } = await safeTry(() => fnC(b));
  if (errC) return handleError('fnC failed', errC);
  
  return { status: true, data: c };
}
```

**✅ Right Way (.then for simple data pipelines):**

```ts
fetch('/api/data')
  .then(res => res.json())
  .then(data => data.items.map(item => ({ 
    id: item.id, 
    name: item.name.toUpperCase() 
  })))
  .then(items => console.log('Processed items:', items))
  .catch(err => console.error('Pipeline failed:', err));
```

**Why This Works:** The first example makes dependencies explicit and gives you precise error handling. The second example treats the entire chain as a data transformation pipeline - clean and functional.

## 6. Modern JavaScript (ES6+) - Why It Matters

This isn't about being trendy - ES6+ features prevent entire categories of bugs and make code more expressive. When you use modern syntax, you're not just writing differently, you're thinking differently.

### The Safety Net

Modern JavaScript gives you tools that prevent common mistakes:

- **`const` and `let`** eliminate variable hoisting confusion and accidental reassignment
- **Arrow functions** remove `this` binding surprises
- **Destructuring** makes data extraction explicit and readable
- **Template literals** prevent string concatenation errors
- **Modules** create clear boundaries between code

### Examples

**❌ Wrong Way (old syntax, hidden bugs):**

```js
var x = 1;  // Can be accidentally reassigned anywhere
function add(a, b) { 
  return a + b;  // What if a or b is undefined?
}
module.exports = { add };  // CommonJS mixing with ES6 imports causes confusion
```

**✅ Right Way (ES6+, explicit and safe):**

```ts
export const add = (a: number, b: number): number => a + b;
// const prevents reassignment
// TypeScript catches type errors
// ES6 modules are consistent
```

**Real Impact:** The old version looks innocent, but `var x` can be reassigned anywhere in your code, the function doesn't validate inputs, and mixing module systems creates import confusion. The new version makes all of these issues impossible.

### Why This Prevents Bugs

**Variable Safety:**

```ts
Old way - var can be redeclared and hoisted
var user = getUser();
if (user) {
  var user = processUser(user);  // Accidentally overwrote original!
}

New way - const/let prevent redeclaration
const user = getUser();
if (user) {
  const processedUser = processUser(user);  // Clear intent, no accidents
}
```

**Function Clarity:**

```ts
Old way - function context confusion
const api = {
  users: [],
  getUser: function(id) {
    return this.users.find(function(user) {  // `this` is now undefined!
      return user.id === id;
    });
  }
};

// ✅ New way - arrow functions preserve context
const api = {
  users: [],
  getUser: (id) => api.users.find(user => user.id === id)  // Clear and predictable
};
```

**The Rule:** Use ES6+ not because it's modern, but because it makes your code safer and more predictable.

## 7. Occam's Razor - The Art of Simplicity

"Among competing solutions, the simplest one is usually correct." This 14th-century principle applies perfectly to code. Every line of complexity you add is debt that future developers will pay with interest.

### Why Choose Simple Over Clever

Complex code might make you feel smart when you write it, but it makes everyone (including future you) feel stupid when they try to understand it. Simple code is a gift to your team and your future self.

**The Trade-off:** Simple solutions might seem less impressive, but they're easier to debug, modify, and reason about. Your goal isn't to showcase your skills - it's to solve problems clearly.

### Examples

**❌ Wrong Way (clever but obscure):**

```ts
// This works, but requires mental gymnastics to understand
const flattened = arr.reduce((a, b) => a.concat(b), []);
```

**✅ Right Way (simple and clear):**

```ts
// Intent is immediately obvious
const flattened = arr.flat();
```

**❌ Wrong Way (monolithic function doing everything):**

```ts
function process(items, config, flags, options, handler) { 
  // 300 lines of mixed concerns
  let results = [];
  for (let i = 0; i < items.length; i++) {
    if (config.enableValidation && flags.strictMode) {
      // validation logic mixed with processing
      if (!items[i].id || items[i].id.length < 3) continue;
    }
    
    let processed = items[i];
    if (options.transform) {
      processed = options.transform(processed);
    }
    
    if (handler) {
      handler.onProcess(processed);
    }
    
    results.push(processed);
  }
  return results;
}
```

**✅ Right Way (simple functions composed together):**

```ts
const validateItem = (item) => {
  return item.id && item.id.length >= 3;
};

const transformItem = (item, transformer) => {
  return transformer ? transformer(item) : item;
};

const processItems = (items, { validator, transformer, handler }) => {
  return items
    .filter(validator || (() => true))
    .map(item => transformItem(item, transformer))
    .map(item => {
      handler?.onProcess(item);
      return item;
    });
};
```

### The Simplicity Test

Before you commit code, ask yourself:

1. **Can a junior developer understand this in 30 seconds?**
2. **If I came back to this in 6 months, would I immediately know what it does?**
3. **Is there a simpler way that solves the same problem?**

If any answer is "no," simplify further.

### When Complex Is Okay

Use complex approaches only when you've measured that the simple approach genuinely doesn't work (performance, memory, etc.) and you document why the complexity is necessary.

**The Rule:** Start simple, add complexity only when you have compelling evidence it's needed, and always document your reasoning.

## 8. Predictability - Making Code Behavior Obvious

Predictable code is code where you can guess what happens before you run it. When code behavior is obvious, debugging becomes trivial and changes become safe.

### The Map/Object Lookup Pattern

One of the most powerful patterns for predictability is replacing long switch statements or if/else chains with object lookups. This makes adding new behavior trivial and the code becomes self-documenting.

### Why This Matters

When you see a function with a long switch statement, you have to mentally trace through all the cases to understand what might happen. When you see an object lookup, you immediately know: "This function will call the handler for the given key, or do nothing if the key doesn't exist."

### Examples

**❌ Wrong Way (switch statement that grows and grows):**

```ts
function handleAction(action, payload) {
  switch(action) {
    case 'create': 
      validateCreate(payload);
      return handleCreate(payload);
    case 'update': 
      validateUpdate(payload);
      return handleUpdate(payload);
    case 'delete': 
      confirmDelete(payload);
      return handleDelete(payload);
    // ... 10 more cases that you have to read through
    case 'archive': 
      return handleArchive(payload);
    default:
      throw new Error(`Unknown action: ${action}`);
  }
}
```

**✅ Right Way (object lookup - behavior is immediately obvious):**

```ts
// Define handlers separately - keeps them focused and testable
const handleCreate = (payload) => {
  validateCreate(payload);
  return createRecord(payload);
};

const handleUpdate = (payload) => {
  validateUpdate(payload);
  return updateRecord(payload);
};

const handleDelete = (payload) => {
  confirmDelete(payload);
  return deleteRecord(payload);
};

const handleArchive = (payload) => {
  return archiveRecord(payload);
};

// Map actions to their handlers - clean and obvious
const actionHandlers = {
  create: handleCreate,
  update: handleUpdate,
  delete: handleDelete,
  archive: handleArchive
};

function handleAction(action, payload) {
  const handler = actionHandlers[action];
  if (!handler) {
    return { status: false, message: `Unknown action: ${action}` };
  }
  return handler(payload);
}
```

**Why Define Handlers Separately:**

- Each handler has a clear, focused purpose
- Handlers are easier to test in isolation
- The mapping object stays clean and readable
- You can reuse handlers in different contexts
- Debugging is simpler when each handler is a named function

### Why This Works Better

1. **Adding new actions** is just adding a new property - no modification of existing code
2. **The behavior is obvious** - it will call the handler for the action or return an error
3. **Testing is easier** - you can test each handler independently
4. **No fall-through bugs** - impossible to accidentally fall through cases

### When to Use Switch vs Object Lookup

**Use object lookup when:**

- You have simple 1:1 mappings of input to behavior
- You expect to add more cases over time
- Each case is independent

**Use switch when:**

- You need complex pattern matching with fall-through
- Cases share logic and you want to group them
- You have complex conditions that can't be expressed as simple keys

**The Rule:** Default to object lookups. Use switch only when you need features that objects can't provide, and document why.

## 9. Control Flow - Switches, Nested Ifs, and When to Use What

Control flow is where code readability lives or dies. The way you structure conditionals determines whether future developers can quickly understand your logic or get lost in a maze of nested conditions.

### The Hierarchy of Control Flow Clarity

From most readable to least readable:

1. **Early returns** (guard clauses) - Best for validation and error handling
2. **Object/Map lookups** - Best for simple 1:1 mappings  
3. **Single-level if/else** - Good for binary decisions
4. **Switch statements** - Acceptable for complex pattern matching
5. **Nested if/else** - Last resort, avoid when possible

### When to Use Each Pattern

**Early Returns (Guard Clauses):**

- Input validation and precondition checking
- Error conditions and edge cases
- When you want the happy path to be unindented and obvious

**Object/Map Lookups:**

- Simple action dispatching
- Configuration-driven behavior  
- When adding new cases should be trivial
- 1:1 mapping of input to output

**Switch Statements:**

- Complex pattern matching with fall-through
- When cases share common logic
- State machine implementations

**Single-level If/Else:**

- Binary decisions
- When logic is too complex for object lookup
- When conditions involve complex expressions

**Nested If/Else (Avoid):**

- Only when all other patterns genuinely don't fit
- Document why nesting is necessary
- Consider refactoring into smaller functions

**The Rule:** Choose the pattern that makes your intent most obvious to someone reading the code for the first time.

## 10. Declarative vs Imperative Programming

Here's a fundamental choice you make every time you write code: should you tell the computer *how* to do something step by step (imperative), or should you describe *what* you want and let built-in methods handle the details (declarative)?

### Why Choose Declarative When Possible

Declarative code expresses intent clearly. When you read `users.filter(user => user.active)`, you immediately understand the goal - get active users. When you read a for loop with conditions and array pushing, you have to mentally execute the code to understand the intent.

**The Trade-off:** Declarative code is usually more readable but sometimes less performant. Imperative code gives you control but requires more mental overhead to understand.

### Examples

**❌ Wrong Way (verbose imperative when declarative exists):**

```ts
let activeUsers = [];
for (let i = 0; i < users.length; i++) {
  if (users[i].active && users[i].role !== 'admin') {
    let transformed = {
      id: users[i].id,
      name: users[i].name,
      email: users[i].email,
      lastLogin: users[i].lastLogin
    };
    activeUsers.push(transformed);
  }
}
```

**✅ Right Way (declarative):**

```ts
const activeUsers = users
  .filter(user => user.active && user.role !== 'admin')
  .map(user => ({
    id: user.id,
    name: user.name,
    email: user.email,
    lastLogin: user.lastLogin
  }));
```

**✅ Acceptable Imperative (when performance matters):**

```ts
const activeUsers = [];
for (const user of users) {
  if (user.active && user.role !== 'admin') {
    activeUsers.push({
      id: user.id,
      name: user.name,
      email: user.email,
      lastLogin: user.lastLogin
    });
  }
}
```

### When to Use Imperative

Use imperative style when you've measured performance and the declarative approach is genuinely too slow, or when the logic is complex enough that breaking it into steps actually makes it clearer. Document your reasoning when you choose imperative for performance.

**The Rule:** Start declarative, move to imperative only when you have a compelling reason.

## 11. Documentation & Testing Philosophy

Documentation isn't about explaining what your code does - good code should be self-explanatory. Documentation is about explaining why you made certain decisions and what the broader context is. Avoid useless commenting and using documentation to cover up poorly written code, if it needs a lot of explaining, it's not that good.

### When to Document

- **Complex business logic:** If there's a formula or algorithm that isn't obvious
- **Architectural decisions:** Why you chose this pattern over alternatives
- **Integration points:** How your code fits into the larger system
- **Error conditions:** What can go wrong and how callers should handle it
- **All reusables documented** All re-usable functions should be documented

### JSDoc for Public APIs

```ts
/**
 * Processes a payment transaction with fraud detection
 * @param transaction - Payment details and customer info
 * @param options - Processing options (test mode, fraud checks, etc.)
 * @returns Result with transaction ID or error details
 */
export const processPayment = async (transaction, options = {}) => {
  // Implementation
};
```

### Testing Philosophy

- **Unit tests** for pure functions and business logic
- **Integration tests** for database operations and external APIs
- **Test error paths** - Don't just test the happy path
- **Use dependency injection** - Makes mocking natural and easy
- **Use creative testing** - Sometimes you need to test in ways that make sense for the scenario. It's okay to not use conventional testing approaches if the situation calls for it.
- **Every re-usable code** should be tested, every critical function and logic too!

For Ts/Js testing vitest is better, linting biome via ultracite is better.

## 12. Task And Spec Driven Development Workflow

Before you write code, write down what you're planning to do and why. This isn't bureaucracy - it's thinking time that saves you from costly mistakes.

### When to Create a Task File

- **New features** or significant changes to existing features
- **Bug fixes** that require investigation or affect multiple files
- **Refactoring** that touches more than a few functions
- **Performance optimizations** or architectural changes
- **Any work** you estimate will take more than 30 minutes
- **Anything Probably Never Seen Before** A new method or algorithm or complex feature, create a feature.spec.md file, a specification file detailing how it will and should work, and get this approved before implementing. Serves as source of truth.

### What Goes in task.md

```markdown
## Title
Brief description of what you're going to build

## Context
Why this change is needed - the business or technical problem you're solving

## Steps
1. Specific action items
2. Dependencies and order of operations
3. Testing approach

## Decisions
Important choices you made and why (library selection, architecture, etc.)

## Status
[ ] Planning
[ ] In Progress  
[x] Complete
```

### Why This Works

Writing forces you to think through the problem before you start coding. Most bugs and scope creep happen because we start implementing before we fully understand what we're building.

Also helps, to create a context.md file that you can keep updating with quick facts as you explore a codebase, this helps you revise later without going through the entire code again, and a plan.md may also be used to organize tasks into epics and phases and pseudo code or implementation plans to complement the task.md file.

## 13. Legacy Code Rules & Process

Legacy code is code without tests, or code you're afraid to change. Here's how to work with it safely:

**Rules:**

- **Ask the owner/author before changing legacy code** - They know the hidden gotchas
- **Don't make assumptions** - Some code may seem like a mistake, until it was intentional, don't make assumptions, find the intent and test behaviour or ask around
- **Write tests that capture current behavior** (even if buggy) or document it down and make file copies if needed
- **Propose change in task.md** - Consider feature flags for transition
- **Preserve semantics unless explicitly changing** - With versioned changes and git
- **Commit at critical points** - Before changing code that's confirmed to work

**❌ Wrong Way:**

- Refactor and merge without tests or approval

**✅ Right Way:**

- Add tests, propose change, implement safely

### Why These Rules Exist

Legacy code often has invisible dependencies and behaviors that aren't documented anywhere except in the code itself. One small change can break something seemingly unrelated. The rules force you to understand before you change. Most of the time, you want to edit code not to rewrite, there's a difference, and most cases find ways that are less likely to break the exposed contract or api how a function is consumed, inner working may change, not the interface unless only adding to it.

## 14. Essential Utilities (Copy-Paste Ready)

### safeTry Implementation

```ts
// Works for both sync and async operations
export function safeTry<T>(fn: () => T): { err: Error | null; result: T | null };
export function safeTry<T>(fn: () => Promise<T>): Promise<{ err: Error | null; result: T | null }>;
export function safeTry<T>(fn: () => T | Promise<T>) {
  try {
    const result = fn();
    if (result instanceof Promise) {
      return result
        .then(value => ({ err: null, result: value }))
        .catch(error => ({ err: error instanceof Error ? error : new Error(String(error)), result: null }));
    }
    return { err: null, result };
  } catch (error) {
    return { err: error instanceof Error ? error : new Error(String(error)), result: null };
  }
}
```

## 16. Pre-Commit Checklist

### Code Quality

- [ ] No classes used?
- [ ] Functions have single responsibility?
- [ ] Error handling with safeTry pattern?
- [ ] Early returns used for validation?
- [ ] File under 400 lines?
- [ ] Names are self-explanatory?
- [ ] ES6+ features used appropriately?

### Architecture

- [ ] Dependencies injected (not imported deep in functions)?
- [ ] Barrel files used for module exports?
- [ ] Types exported separately if reused?
- [ ] No circular dependencies?

### Documentation

- [ ] Complex logic has JSDoc comments?
- [ ] task.md updated with decisions?
- [ ] Breaking changes documented?

## 18. Glossary

### Technical Terms

**ADT (Algebraic Data Type):** A data structure that can be one of several distinct shapes. For example, a Result type that's either success OR failure, never both.

**API (Application Programming Interface):** A set of rules and tools for building software applications. Defines how different software components should interact.

**Barrel File:** An `index.ts` file that exports all the public functions from a module, providing a clean entry point.

**Closure:** A function that has access to variables from its outer (enclosing) scope even after the outer function has returned.

**Composition:** Building complex functionality by combining simple, focused functions rather than creating large, monolithic functions.

**DI (Dependency Injection):** A technique where dependencies are provided to a function/module from the outside rather than created inside.

**Factory Pattern:** A design pattern that creates objects (or in our case, function collections) without specifying the exact implementation details.

**HOF (Higher-Order Function):** A function that takes other functions as arguments or returns a function as a result.

**Imperative Code:** Code that describes HOW to do something, step by step (like a recipe).

**Declarative Code:** Code that describes WHAT you want to achieve, without specifying the exact steps.

**LOC (Lines of Code):** A measure of code length. We limit files to 400 LOC for maintainability.

**OOP (Object-Oriented Programming):** A programming paradigm based on objects and classes. We avoid this in favor of functional programming.

**Pure Function:** A function that always returns the same output for the same input and has no side effects.

**Side Effects:** Operations that affect things outside the function scope (like modifying global variables, making API calls, or writing to files).

### Acronyms

**DRY:** Don't Repeat Yourself
**KISS:** Keep It Simple, Stupid  
**YAGNI:** You Aren't Gonna Need It
**SRP:** Single Responsibility Principle
**SOLID:** Single responsibility, Open/closed, Liskov substitution, Interface segregation, Dependency inversion

## 20. Final Notes

This is a work in progress and subject to change and nothing herein is set in stone, suggestions are always welcome but if no convincing reason, the rules here are almost laws!

**Author:** [Hussein Kizz](https://github.com/Hussseinkizz) at Nile Squad Labz

*This specification reflects the current implementation and is subject to evolution. Contributions and feedback are welcome.*
