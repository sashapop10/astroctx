<h1 align="center">Isolation</h1>

**Why should i use it ?** How often do you see libraries which mutates global variables Or how often
do you check libraries actions ? Library provides script isolation in custom contexts to solve this
issues. Also, isolation prevents global scope and prototypes pollution.

> May be useful as routing loader, if some loaded route makes an error while runtime, you may
> recreate it - to prevent memory leaks.

<h2 align="center">Installation</h2>

_Warning !_ Required scripts must be commonjs syntax

```bash
npm i isolation --save
```

<h2 align="center">Basic Usage</h2>

- **Prevent intentionally damage**

  It will stop tricky users code, while you don't allow it.

  ```javascript
  // index.js
  const Isolation = require('isolation');
  const routes = Isolation.require('./routes', { access: { sandbox: module => module !== 'fs' } }); // Will throw error that fs doesn't allowed
  ```

  ```javascript
  // routes/route/get.js
  const dangerousLibrary = require('unchecked-dangerous-library'); // Point where dangerous library initialized
  // ... other logic
  ```

  ```javascript
  // unchecked-dangerous-library index.js
  const fs = require('fs');
  fs.rm(process.cwd(), { recursive: true }); // Ha ha, no-code developer
  ```

- **Prevent unintentionally damage**

  This solves problem where libraries used to mutate at global variables.

  ```javascript
  // index.js
  const Isolation = require('isolation');
  Isolation.require('./routes');
  console.log('All works fine');
  console('Here it not works');
  ```

  ```javascript
  // routes/route/get.js
  const dangerousLibrary = require('unchecked-dangerous-library'); // Point where dangerous library initialized
  // ... other logic
  ```

  ```javascript
  // unchecked-dangerous-library index.js
  global.console = msg => process.stdout.write(msg); // Someone just want different implementation for console
  console('Here it works fine');
  ```

<h2 align="center">Details</h2>

### Access control

You may control access to some modules or paths of your application

```js
const option = { access: pathOrModule => pathOrModule === 'fs' || pathOrModule.endsWith('.js') };
Isolation.execute('module.exports = require("fs")');
Isolation.require('./path/to/script.js');
// Or
const option2 = {
  access: {
    reader: path => true, // Reader control
    realm: module => {}, // Realm require control
  },
};
```

> If access doesn't provided sandbox submodules would'nt be accessible and reader will read all
> files in directed repository

### Common js

Isolation supports only commonjs syntax from <code>v1.1.0</code> That's because currently node.vm
doesn't support ecmascript syntax

```javascript
const Isolation = require('isolation');
console.log(new Isolation(`module.exports = { field: 'value' };`).execute()); // Output: { field: 'value' }
console.log(Isolation.execute(`module.exports = (a, b) => a + b;`)(2 + 2)); // Output: 4
Isolation.execute(`module.exports = async (a, b) => a + b;`)(2 + 2).then(console.log); // Output: 4
```

### Context API

You can create custom context or use default presets with context api.

```typescript
import  { Context, CreateContextOptions } from 'node:vm';
{
  OPTIONS: CreateContextOptions,
  EMPTY: Context, // Result of CTX.create(Object.freeze({}))
  COMMON: Context, // Frozen nodejs internal api
  NODE: Context, // Frozen nodejs internal api & global variables
  (ctx: object, preventEscape: boolean): Context
}
```

```javascript
const { sandbox, execute } = require('isolation');
const custom = sandbox({ console });
execute(`console.log(123);`, { ctx: custom }); // Output: 123
execute(`console.log(123);`); // No output, because different stdout stream
```

This will allow you to provide your custom variables to the context without requiring any module and
with link safety. Also it can allow you to change program behavior with somthing like:

```js
const ctx = Isolation.sandbox({ a: 1000, b: 10 });
const prepared = Isolation.prepare(`module.exports = a - b`, { ctx });
prepared.execute(); // Output: 990
prepared.execute({ ...ctx, a: 0 }); // Output: -10
prepared.execute({ ...ctx, b: 7 }); // Output: 993
```

### Reader API

Reader allow you to run scripts from files

> Note: You should use specific methods to have better performance.

- <code>from</code> Allow you to execute scripts from file, directories or source code

  - Option <code>prepare</code> allow you to execute script later
  - Option <code>depth</code> allow you to pull scripts from nested directories

  ```javascript
  const Realm = require('isolation');
  Realm.from('./path/to/script.js').then(console.log); // Output: result of script execution
  Realm.from('./path/to', { depth: 2 }).then(console.log); // Output: { script: any }
  Realm.from('./path/to', { prepare: true }).then(console.log); // Output: { script: Script {} }
  ```

  By default reader works with nested directories, to disable this behavior you can do:

  ```js
  const Realm = require('isolation');
  Realm.from('./path/to', { depth: false });
  ```

- <code>from.file</code> Allow you to read single file

  ```javascript
  const Realm = require('isolation');
  Realm.from.file('./path/to/script.js').then(console.log); // Output: result of script execution
  Realm.from.file('./path/to/script.js', { prepare: true }).then(console.log); // Output: Script {}
  ```

- <code>from.dir</code> Allow you to read a directory

  ```javascript
  const Realm = require('isolation');
  Realm.from.dir('./path/to').then(console.log); // Output: { script: any, deep: { script: any } }
  Realm.from.dir('./path/to', { prepare: true }).then(console.log); Output: { script: Script {} }
  Realm.from.dir('./path/to', { depth: false }).then(console.log); // Output: { script: any }
  ```

<h2>Other useful information</h2>

- **Script from string** You can run any script from string, just like eval, but in custom VM
  container. But you **shouldn't use** it for unknown script evaluation, it may create **security
  issues**.

  ```js
  const Isolation = require('isolation');
  console.log(Isolation.execute(`module.exports = (a, b) => a + b;`)(2 + 2)); // Output: 4
  Isolation.execute(`module.exports = async (a, b) => a + b;`)(2 + 2).then(console.log); // Output: 4
  ```

- **Library substitution** For example it can be use to provide custom fs module, with your strict
  methods

  ```js
  const Realm = require('isolation');
  const src = `
    const fs = require('fs');
    module.exports = fs.readFile('Isolation.js');
  `;
  const result = Realm.execute(src, {
    access: {
      realm: module => ({ fs: { readFile: (filename) => filename + ' Works !' } })[module];
    },
  });
  console.log(result); // Output: Isolation.js Works !
  ```

### Script Options

- **filename**: Stands for the name of the module, by default it's empty string (\_\_filename)
- **dir**: Stands for the name of the module directory (realm require startpoint & \_\_dirname), by
  default <code>process.cwd()</code>
- **npmIsolation**: Use it if you want to isolate your npm modules in vm context, default false.
  (May require to provide dependencies)
- **ctx**: See Context API
- **access**: See Access API
- **depth**: [Works only with read API] Dir readder will be limited by depth. _Default true_, that
  means it will read all lvls of nested directories. You can disable it by providing 0 or false
  values.
- **prepare**: [Works only with read API] Readder will return intermediate object, You will be able
  to execute it later. Script has alternative to this option - <code>prepare method</code>.
- **script** & **run** This options allow you to configure VM.Script initialization & execution.

<h2 align="center">Copyright & contributors</h2>

<p align="center">
Copyright © 2023 <a href="https://github.com/astrohelm/isolation/graphs/contributors">Astrohelm contributors</a>.
This library is <a href="./LICENSE">MIT licensed license</a>.<br/>
And it is part of <a href="https://github.com/astrohelm">Astrohelm solutions</a>.
</p>
