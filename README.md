# Prototype Pollution Mitigation / Symbol.proto

**Authors**: [Santiago Díaz](https://github.com/salcho) (Google), [Jun Kokatsu](https://github.com/shhnjk) (Google)

**Champion**: Shu-yu Guo (Google)

**Stage**: 0

# TOC

1. [Problem and motivation](#problem-and-motivation)
1. [Proposal overview](#proposal-overview)
1. [Secure mode](#secure-mode)
1. [New symbols](#new-symbols)
1. [Adoption considerations](#adoption-considerations)

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

In secure mode, it is no longer possible to reach the prototype using string property keys.

There are two alternatives:

- **Option 1**: Delete the `__proto__` and `prototype` keys
- **Option 2**: Delete the `__proto__` and `constructor` keys

Existing codebases that use either of these properties and that opt into secure mode must be refactored to use alternatives. Secure mode forces codebases to get an explicit prototype reference with `getPrototypeOf` before making any changes to it.

An important design question is what should be the mechanism to opt into secure mode. Some alternatives for this include:

- **Introduce an out-of-band mechanism to enable secure mode. For example, in the form of an HTTP header.**
    - This option allows applications to enable/disable secure mode at any time without code changes and to roll out this feature gradually to their users, in a way that avoids breakages. This option is in line with other web platform mitigations.
- **Introduce a `use secure-mode` directive or an `Object.enableSecureMode()` API that instructs JS engines to delete the prototype properties.**
    - These options must be called as early as possible at runtime, because they change the semantics of accessors to prototypes.
- **Enable secure mode when Symbol.proto/constructor is used at least once.**
    - JS engines could enable secure mode when a piece of code makes use of the new symbols (see below). This would allow services to enable secure mode seamlessly, but it would not give them the ability to opt out without making production changes.

Applications would also only be fully compatible once their codebase and all of their dependencies are refactored to use these symbols. Note that this hurdle can be fixed with automatic refactoring.

# New Symbols

Some codebases that opt into secure mode will still need to make changes to prototypes dynamically. To preserve backward compatibility for these cases, we propose creating a new symbol that can support these use cases. Unlike string keys, symbol values are harder to exploit. We propose two alternatives:

## Option 1: `Symbol.proto`

Applications need access to both the internal `[[Prototype]]` slot and the `prototype` property. Since secure mode removes the `__proto__` and `prototype` properties, we will need to keep this parity. In secure mode, applications will get access to both of these by:

- Using `Object.getPrototypeOf` and `Reflect.getPrototypeOf`, which already give access to the `[[Prototype]]` internal slot, as described in this spec section.
- Using `Symbol.proto` to get access to the `prototype` property, which corresponds to this spec section for Object, and equivalents for other types.

`__proto__` and `prototype` can differ in certain cases, which is why this symbol is needed.

Examples:

```javascript
const obj = {};
obj[Symbol.proto] === undefined; // true because obj doesn't have a `prototype` prop.

const arr = [];
arr[Symbol.proto] === undefined; // true because arr doesn't have a `prototype` prop.

function Foo() {}
Foo[Symbol.proto] === Foo.prototype; // true
Foo[Symbol.proto] === Foo.__proto__; // false because this is the [[Prototype]] prop.

Object[Symbol.proto] === Object.prototype; // true
Object[Symbol.proto] === Object.__proto__; // false because this is [[Prototype]]
```

## Option 2: `Symbol.constructor`

In this alternative, codebases can still access the `prototype` property, but they can no longer access the `constructor` property. Any existing accesses to `constructor` will need to be replaced by `Symbol.constructor` or an equivalent mechanism (e.g. `Object.getConstructorOf`).

This alternative provides the same security guarantees as option 1 because, apart from `__proto__`, exploits can only get a hold of prototypes through `object.constructor.prototype`. If the constructor property is no longer accessible, exploits of those vulnerabilities will break.

The constructor property is part of some algorithms spec'd in ECMAScript, for example those in typed arrays. Under this proposal, the spec will need to be changed so that those algorithms can continue to work even when secure mode removes the constructor property.

# Adoption considerations

## Making codebases secure by default with automatic refactoring

The secure mode and symbol pair allows the majority of applications to eliminate prototype pollution without any code changes, because JS parsers/lexers can apply drop-in replacements for any code. Any JS parser/lexer, say in browser engines, can do this in three steps:

1. Find all **static** references of `__proto__` or `prototype`, for example `object.__proto__` or `Object.prototype`
1. Replace each reference with its drop-in, secure alternative. For example, `Object.getPrototypeOf(object)` or `Object[Symbol.proto]`
1. Delete the `__proto__` and `prototype` properties.

This same algorithm applies for option 2, but in this case it is `constructor` references that get replaced. This means that codebase can continue using prototype/constructor properties transparently.

As JS engines and browsers are able to refactor applications on the fly to make them compatible with secure mode, there is potential to make codebases secure by default. We refer to this as automatic refactoring, which allows developers to opt into secure mode without making changes to their codebases, thereby securing them by default. This feature will remove most blockers for adoption, allowing legacy applications to get the security benefits with little to no engineering efforts.

A benefit of implementing this proposal on the browser (i.e. automatic refactoring) is that the proposed symbols don't need to be exposed to developers, making them effectively unutterable.

## Need for supporting opting out of secure mode?

We expect the majority of codebases to be able to opt into secure mode without any code changes, since dynamic modifications to prototypes are not common. If this proposal is implemented without automatic refactoring, a small percentage of codebases will need to be refactored to be compatible with secure mode, by using one of the symbols above.

If a codebase that has been modified to use the new symbols wants to opt out of secure mode, the symbols will have to continue working as regular accessors to the prototype/constructor. This is to keep backward compatibility and give developers the flexibility to enter/exit secure mode without code changes.

Why would a codebase want to opt out of secure mode? Occasionally, mitigations in the web platform need to be rolled back because bugs or regressions in browsers can cause applications to break. Also, if secure mode is not fully compatible with either application code or third party dependencies, developers will want to revert opting into it.

If this breakage potential doesn't apply to JS engines and opting out isn't a use case that needs to be explicitly supported, then the new symbols can be made such that they are only valid accessors in secure mode.
