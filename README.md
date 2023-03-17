
# Prototype Pollution Mitigation / Symbol.proto

**Authors**: [Santiago Díaz](https://github.com/salcho) (Google), [Jun Kokatsu](https://github.com/shhnjk) (Google)

**Champion**: Shu-yu Guo (Google)

**Stage**: 0

# TOC

1. [Problem and motivation](#problem-and-motivation)
2. [Proposal overview](#proposal-overview)
3. [Secure mode](#secure-mode)
4. [Opting into secure mode](#opting-into-secure-mode)
5. [New symbols](#new-symbols)
6. [Adoption considerations](#adoption-considerations)

# tl;dr

This proposal seeks to mitigate prototype pollution by introducing an opt-in secure mode that makes prototypes impossible to access using string property keys, instead requiring they be accessed with methods (`Object.getPrototypeOf`) or the proposed new symbol property keys.

# Problem and Motivation

## What is prototype pollution?

We want to solve prototype pollution in all JS environments: a vulnerability class in JavaScript that allows attackers to manipulate objects they don't control or don't have access to at runtime. This 'spooky action at a distance' primitive can be used to change the shape of other objects and override their properties, thereby tainting objects in the runtime. Tainted objects invalidate the underlying assumptions of code that would otherwise be safe/correct, which creates security issues such as XSS, RCE, and logic bugs in JavaScript applications in any JS environment. Prototype pollution bugs are not limited to web applications.

Pollution vulnerabilities exist because JS properties can be changed by default by anyone who has a reference to them. In particular, if a caller is allowed to modify a property that is shared by many objects, like the properties in the prototype, then that caller can effect changes on other objects, even without having a reference to them.

For example:

```javascript
// source is attacker-controlled
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === 'object')  {
      if(target[key] === undefined) {
        target[key] = {};
      }
      target[key] = merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

const userSuppliedObj = JSON.parse('{"__proto__": {"polluted": true}}');
merge({}, userSuppliedObj); // Prototype Pollution!
const newObj = {};
console.log(newObj.polluted); // true
```

Pollution bugs are a type of data-only attacks that lie outside of the threat model of many existing mitigations, like the Content Security Policy.

## Intentional vs unintentional prototype modifications

Prototypes can be accessed, like any property, using dot and bracket notation. This can have unexpected consequences: for example, a developer wanting to set a property on an object might write the assignment `obj[kOne][kTwo] = value`, where `kOne` and `kTwo` are controlled by the user.

By doing so, they are unintentionally allowing the statement to evaluate to `obj['__proto__']['attackerProperty'] = 'attackerValue'` at runtime, causing all objects that have this prototype in their chain to have `attackerProperty` set on them. This is in contrast with assignments of the form `Object.prototype.newProperty = someFunction()` that make intentional changes to the prototype and are unlikely to mix user input.

By making prototypes available as regular properties, JS code is prone to prototype pollution vulnerabilities that abuse functions that use bracket notation, where the keys in the brackets are not known at compile time. This leads to those functions unintentionally modifying prototypes.

Intentional modifications to prototypes often involve dot notation, e.g. `object.prototype` and `object.__proto__`. whereas statements that use dynamic access of the form `object[key]` can lead to unintentional modifications of prototypes. JS doesn't differentiate between dot and bracket notations and so it can't distinguish between intentional and unintentional changes.

This introduces a symmetry between intentional/unintentional changes, static/dynamic accesses and dot/bracket notation that runs deep in JS. This symmetry is not limited to the language spec but also to JS engines and their internal optimizations for property access, as well as external tools such as code minifiers. This makes it hard to differentiate intentional accesses from unintentional accesses within the JS engine without major changes.

If functions were required to get an explicit reference to the prototype before making changes to it, it would be significantly more difficult to write code that changes prototypes unintentionally. Any method that can distinguish between intentional and unintentional modifications will effectively solve prototype pollution. Conversely, any mitigation that fails to differentiate between intentional (static) and unintentional (dynamic) changes to prototypes will not be backwards compatible, since code that makes intentional changes to prototypes will stop working. An example of this is `Object.freeze`.

## Problems with existing mitigations

Existing solutions against unintentional changes to prototypes like `Object.freeze`, `preventExtensions` and `seal` have a number of downsides that make them difficult to deploy:

- The override mistake and [other inconsistencies](https://docs.google.com/presentation/d/17dji3YM5_LeMvdAD3Y3PQoXU1Mgm5e2yN_BraBSTkjo/edit) in walking the prototype chain to check for writable properties affect `freeze`.
- Many applications and libraries rely on adding functions to prototypes to augment built-in types (i.e. polyfills), which is prohibited by `freeze`, `preventExtensions`, and `seal`, effectively breaking existing applications and blocking adoption.
- File size concerns, where if an application wants to protect its own prototypes (for example `(User|MyClass|Settings).prototype`), they must make one call per type. Application types are usually in the hundreds or thousands, which makes this option verbose and difficult to maintain.
- A hard requirement on strict mode to avoid silent failures that lead to unexpected breakages that are difficult to debug when prototypes are changed.

## Recent vulnerabilities in JS environments

Google has seen an upward trend in bugs submitted to our Vulnerability Rewards Program: 1 in 2020, 3 in 2021 and 5 so far in 2022. We have identified several more in our internal research.

Example vulnerabilities include:

1. **On the Web**: Several XSS issues in services that should have been protected because they use [Strict CSP](https://w3c.github.io/webappsec-csp/#strict-csp). And a [wide range of known vulnerable libraries](https://github.com/BlackFan/client-side-prototype-pollution).
1. **On the desktop**: An bug in a Google-owned desktop application where users could be given a malicious JSON object that could allow local files to be leaked due to a pollution vulnerability. (Currently non-public, disclosure TBD.)
1. **In security features**: Multiple bypasses in sanitizers, including [Chrome's Sanitizer API](https://crbug.com/1306450), [DOMPurify and the Closure sanitizer](https://research.securitum.com/prototype-pollution-and-bypassing-client-side-html-sanitizers/).
1. **In the browser**: A [Firefox sandbox escape](https://www.zerodayinitiative.com/blog/2022/8/23/but-you-told-me-you-were-safe-attacking-the-mozilla-firefox-renderer-part-2) leading to remote code execution.
1. **In NodeJS**: Several [RCE](https://research.securitum.com/prototype-pollution-rce-kibana-cve-2019-7609/)s have been discovered.

We expect the number of vulnerable applications will grow as JavaScript applications are deployed to more environments (e.g. Electron, Cloudflare Workers, etc). Therefore, a language-level solution is required to mitigate attacks in all environments.

# Proposal overview
In the spirit of preventing unintentional changes to prototypes via dynamic access, this proposal puts forward two new concepts that complement each other and are meant to be adopted together: an opt-in secure mode that forbids code from referencing prototypes through dynamic access with string keys and a symbol that can be used by specialized applications to change prototypes dynamically in a secure way.

Secure mode achieves two goals: it prevents any vulnerable code (for example, `deepCopy`/`deepMerge` functions) from unknowingly making changes to prototypes and it allows code that intentionally modifies prototypes (e.g. polyfills, reflection-heavy framework code) to continue working.

# Secure mode
In secure mode, it is no longer possible to reach the prototype using string property keys. This can be achieved by always providing secure alternatives like `Symbol.instanceProto` and `Symbol.ctor` and, when secure mode is enabled, deleting the __proto__ and constructor properties. Read more about these in the _New Symbols_ section.

Secure mode is **not available** by default due to breakage potential in a small number of codebases that rely on computed access to prototypes and constructor. Whenever secure mode is enabled, it is applied globally to the JS runtime.

This proposal introduces the concept of parse-time refactoring to maximize compatibility with as many codebases as possible, without requiring manual refactoring from them. This can be achieved by making JS engines rewrite all static usages (`object.__proto__` or `object.constructor`) to their refactoring counterparts (`object[Symbol.instanceProto]` or `object[Symbol.ctor]`) during the parsing phase. As a result, secure mode should only have breakage potential in codebases that rely on computed access, for example `obj[key]` where key is expected to be `__proto__` at runtime. Read more about this in the _parse-time automatic refactoring_ section.

**Note on naming**: 'Secure mode' can be misinterpreted to mean 'all vulnerabilities have been removed' rather than 'this codebase is prototype pollution-free'. A more appropriate name for this mode should be chosen.

# Opting into secure mode

An important design question is what should be the mechanism to opt into secure mode. We recommend introducing an out-of-band mechanism, for example in the form of an HTTP header or command line flag. This option allows applications to enable/disable secure mode at any time without code changes and to roll out this feature gradually to their users, in a way that avoids breakages. This option is in line with other web platform mitigations, like CSP.

Out of band triggers have the added benefit that all decisions needed to enable secure mode can be made before the JS runtime is created, effectively working around the 'freezing point' issues mentioned in the _Problems with existing mitigations_ section.

## Alternatives considered:

-   Introduce a `use secure-mode` directive or an `Object.enableSecureMode()` API that instructs JS engines to delete the prototype properties. This option is significantly more complex than out-of-band opt-ins, because it implies secure mode can be triggered at an arbitrary time during the runtime's lifecycle.
    
-   Enable secure mode when `Symbol.proto`/`constructor` is used at least once. This option is not compatible with bundled assets where a third party library opts into secure mode, but the rest of the code isn't compatible with it.

# New Symbols

We propose creating new symbols that can be used to provide a well-lit path to refactor codebases into being compatible. We choose symbols because, unlike string keys (which are often the type of data in which user input comes from), getting a hold of symbol values in an exploit is significantly more difficult. Exploits will require either additional vulnerabilities that can be chained together or some form of code execution. 

In total, two symbols are needed to stop object instances from reaching their prototypes. These symbols are simple accessors and as such should also be available when secure mode isn't enabled. This allows codebases that have been refactored to use the new symbols to continue working as intended and be backward compatible, either with browsers that don't support secure mode or because the opt in header is not present.

## Symbol.instanceProto

This symbol is a drop in replacement for `__proto__`. Its name indicates that it is different from the prototype property.

## Symbol.ctor

This symbol is a drop in replacement for the `constructor` property.

# Adoption considerations

## `IsConstructor` in other specifications

Unfortunately, there are several specifications that rely on the [IsConstructor](https://tc39.es/ecma262/#sec-isconstructor) definition. This adds friction to removing the `constructor` property because even if an application's codebase doesn't use `constructor` anywhere, usages of the constructor property can still happen at runtime. 

Examples of these are [point 3](https://tc39.es/ecma262/#sec-arrayspeciescreate) of the `ArraySpeciesCreate` algorithm, which references the `constructor` property explicitly or [point 1](https://html.spec.whatwg.org/multipage/custom-elements.html#element-definition) of the custom elements in HTML. 

## Parse-time automatic refactoring

To maximize the number of codebases compatible with secure mode, we propose an algorithm to be implemented in browsers and other runtimes when secure mode is enabled. This algorithm should run during the JS parsing phase:

1.  Find all static references to `__proto__`  and `constructor`, for example `object.__proto__`.
    
2.  Replace each of those references with its drop-in, secure alternative. For example, `object[Symbol.instanceProto]` or `object[Symbol.ctor]`
    
After these drop-in replacements, only code locations that access these properties dynamically will fail, e.g. `object[key]` where `key` is `__proto__` at runtime.

While this refactoring isn't enough to enable secure mode by default for all applications, it significantly lowers the bar for adoption, as it allows developers to focus solely on cases that are, by definition, potentially vulnerable. The process of refactoring those cases is, in itself, a way to remove patterns that lead to pollution vulnerabilities.

## Computed access in minimized JS

Some toolchains produce JS bundles that leverage computed access to minimize the size of the bundle. This is incompatible with parse-time automatic refactoring because it hides accesses to the `__proto__` or `constructor` using static analysis. We have queried the [HTTP Archive](http://httparchive.org) to get an estimate as to how common this pattern is. The following table shows that pages with this behavior are consistently below 1% throughout the last 12 months for all pages crawled with a desktop browser:

| Table              | Documents accessing `__proto__` or `constructor` dynamically | Total number of crawled documents | Ratio |
|--------------------|--------------------------------------------------------------|-----------------------------------|-------|
| 2023_03_01_desktop |                                                    5,407,936 |                       609,469,458 | 0.89% |
| 2023_02_01_desktop |                                                    4,842,383 |                       549,089,708 | 0.88% |
| 2023_01_01_desktop |                                                    5,283,826 |                       589,519,160 | 0.90% |
| 2022_12_01_desktop |                                                    5,161,471 |                       577,073,883 | 0.89% |
| 2022_11_01_desktop |                                                    5,023,169 |                       561,726,239 | 0.89% |
| 2022_10_01_desktop |                                                    4,393,377 |                       476,880,624 | 0.92% |
| 2022_09_01_desktop |                                                    4,239,257 |                       466,278,762 | 0.91% |
| 2022_08_01_desktop |                                                    4,259,814 |                       463,784,047 | 0.92% |
| 2022_07_01_desktop |                                                    3,011,137 |                       339,468,615 | 0.89% |
| 2022_06_01_desktop |                                                    2,301,317 |                       257,501,222 | 0.89% |
| 2022_04_01_desktop |                                                    2,368,577 |                       263,144,657 | 0.90% |
| 2022_03_01_desktop |                                                    2,319,518 |                       259,249,013 | 0.89% |
