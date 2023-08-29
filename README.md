
# Prototype Pollution Mitigation / Symbol.proto

**Authors**: [Santiago Díaz](https://github.com/salcho) (Google)

**Champion**: Shu-yu Guo (Google)

**Stage**: 1

# TOC

* [Problem Description](#problem-description)
  * [Spooky action at a distance](#spooky-action-at-a-distance)
  * [Data-only attacks](#data-only-attacks)
* [Issues with `freeze`, `seal` and `preventExtensions`](#issues-with-freeze-seal-and-preventExtensions-)
  * [The override mistake](#the-override-mistake)
  * [Coarse granularity](#coarse-granularity)
  * [Freezing points](#freezing-points)
  * [Application types](#application-types)
* [Proposed solution](#proposed-solution)
  * [Provide reflection APIs](#provide-reflection-apis)
  * [Opt-in feature](#opt-in-feature)
    * [Automatic refactoring](#automatic-refactoring)
  * [What does delete mean?](#what-does-delete-mean)
* [Incompatible code bases](#incompatible-code-bases)
* [Appendix](#appendix)
  * [What about constructor pollution?](#what-about-constructor-pollution)
  * [Computed access in minimized JS](#computed-access-in-minimized-JS)
  * [Example vulnerabilities](#example-vulnerabilities)

# tl;dr

This proposal seeks to mitigate a language-level vulnerability known as prototype pollution with a mechanism that complements freeze primitives and a mechanism to make most code bases compatible with it. It describes an opt-in feature that makes prototypes available only through reflection APIs. By doing so, the statement `obj[key]` can't access prototypes anymore. Code bases compatible with this feature are more _intentional_ about the way they use prototypes.

# Problem Description

## Spooky action at a distance

PP vulnerabilities allow attackers to manipulate objects they don't control or don't have access to at runtime. This 'spooky action at a distance' primitive can be used to change the shape of other objects and override their properties, thereby tainting objects in the runtime.

Tainted objects invalidate the underlying assumptions of code that would otherwise be safe/correct and can lead to arbitrary code execution and a wide range of other security issues in JS code bases. Prototype pollution bugs express themselves often in web applications, but also affect non-web JS runtime.

Object properties in JS are writeable by any code that can reference them. In particular, if many objects rely on a shared property, any one of them can enact changes on all others.

## Data-only attacks

A special property of PP is that it is a data-only attack, allowing code execution to be attained purely through data. For instance, see the following vulnerable code and a corresponding exploit:

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

// User input comes as a string
const userSuppliedObj = JSON.parse('{"__proto__": {"polluted": true}}');
// Trigger prototype pollution
merge({}, userSuppliedObj);
// Create a brand new object
const newObj = {};
// Has polluted property
console.log(newObj.polluted); // true
```

Note that the exploit is able to taint the creation of new objects _without injecting any foreign code_. 

Because of this special property, modern mitigations against code execution issues -like the Content Security Policy or Trusted Types- fall short of protecting against PP, as they focus on enforcing _code provenance_.

**Note** that data-only attacks are relevant to situations where the code running on the VM is trusted and arbitrary code execution has security impact.

# Issues with `freeze`, `seal` and `preventExtensions`

Existing freezing primitives suffer from significant design issues that make them unlikely to be widely adopted. They can be useful to expert users, but are not suitable to be deployed by the majority of developers, who **reasonably expect prototypes to be mutable**:

### The override mistake

Freeze APIs suffer from the override mistake and [other inconsistencies](https://docs.google.com/presentation/d/17dji3YM5_LeMvdAD3Y3PQoXU1Mgm5e2yN_BraBSTkjo/edit) which introduce bugs in existing code bases, making them throw or worse, _silently fail_ in sloppy mode. A [previous investigation](https://github.com/tc39/ecma262/pull/1320) of the override mistake concluded that the override mistake triggers on [~10% of code bases in strict mode](https://chromestatus.com/metrics/feature/timeline/popularity/2610) and [20% in sloppy mode](https://chromestatus.com/metrics/feature/timeline/popularity/2609). The investigation was dropped shortly after.

### Coarse granularity

Freeze APIs give developers the heavy responsibility of knowing which prototypes should be frozen to maintain a secure code base, assuming that developers are security experts. These APIs describe the _what_ but not the _how_ of security. Freezing `Object` is certainly not good enough, as many exploits abuse `Array`. What about `Error`, `Date`, `Reflect` or `Proxy`? Or future built-in types? Freeze APIs provide no answers to these questions.

### Freezing points

Freeze APIs assume a stable freezing point: a fixed moment at runtime where prototypes have settled and can be frozen. In practice, this point is volatile and changes over time in code bases that are actively developed. While one can find such a point in many applications today, the addition of new dependencies, polyfills, code structure changes and power features like hotswapping and developer tools make freezing points a moving target.

### Application types

Freeze APIs can't protect the full prototype chain. In JS, objects can be added or removed from the prototype chain at any point in time. To protect the full chain, one should always remember to freeze objects that are added to the chain, an error-prone process. When they are removed from the chain, they cannot be made unfrozen anymore.

# Proposed solution

In a nutshell: **a feature that exposes prototypes only to reflection APIs**. If prototypes weren't made available through properties like `__proto__` or `prototype`, they would not be exposed to data-only issues. 

This is better understood through an example: the statement `obj[one][two] = value` is vulnerable to PP through `obj.__proto__.polluted`. If one deletes the `Object.prototype.__proto__` property, the same statement is no longer vulnerable because it can't fit the only other way to reach prototypes, which is `obj.constructor.prototype.polluted`. Note that prototype can't be deleted.

This proposal can be implemented by providing **reflection APIs** and creating a new **opt-in encapsulation feature** that **deletes prototype properties**. A description of each step follows.

### Provide reflection APIs

`__proto__` is a legacy property name that can be deleted, but the internal slot behind it can still be read through `Object/Reflect.getPrototypeOf` and written through `Object/Reflect.setPrototypeOf`, which will simply continue to make this property accessible to code that is already running.

We propose the creation of new APIs for `prototype`, for example `getClassPrototypeOf` and `setClassPrototypeOf`, which would allow this property name to be deleted without changing in any way how this special property works and supports the VM.

Reflection APIs can be polyfilled, which allows hardened code bases to work in all browsers, including older versions.

### Opt-in feature

A new opt-in 'encapsulation feature' where no property names are created for the getter and setter function of prototype slots, which is now possible because references to those properties can use reflection APIs instead. 

The feature is enabled through an **out-of-band flag**:

- In **browser contexts**, through an HTTP header like `X-Encapsulate-Prototype: true`
- In **other contexts**, through a feature flag like `--encapsulate-prototype`

When encapsulation is _disabled_, prototypes are available via both properties and reflection APIs.

When encapsulation is _enabled_, prototypes are only available via reflection APIs, having deleted both `__proto__` and `prototype`.

Encapsulation also includes the following **automatic refactoring** feature:

### Automatic refactoring

When encapsulation is enabled, JS engines loading new source code enable an extra step in their parse phases that registers all dot-notation to prototype properties as if they were calls to their reflection APIs. This step can be implemented efficiently and allows code bases with **third-party, transitive or dynamically-loaded dependencies to be compatible with encapsulation**.

In the future, this change will pave the way for marking `prototype` as deprecated.

### What does delete mean?

Prototype properties could simply be `undefined` when encapsulation is enabled, but they could throw an error when there are attempts to read/write them. This would mean failing faster and loud and would allow migrations to the reflection APIs/encapsulation to be tested.

This implies making the getters and setters of `__proto__` and `prototype` conditional on encapsulation, by using a host hook, in the same way the `eval` function throws under Content Security Policy.

# Incompatible code bases

Code that _relies on computed property access_ to reference prototypes is not compatible with encapsulation or with automatic refactoring. It must be refactored to explicitly refer to prototypes when they are used. This refactoring actually makes the code express intent, which makes dangerous patterns visible to static analysis. In practice, code bases with this characteristic are usually reflection frameworks, debugging tools and other reflection-heavy use cases that are most likely aware of how they use prototypes.

Code bases that use the word `prototype` to define custom properties are not compatible. Such code bases can be made compatible with  encapsulation if this property is always set/get through bracket notation. Historically and based on HTTP Archive queries, there is a small percentage of code bases that are incompatible for this reason.

# Appendix

### What about constructor pollution?

Some changes to the `constructor` property can also have spooky action at a distance. During our research, we have found no practical vulnerabilities affected by this. 

The bar is significantly high for this attack to work: Like in PP, one must find an application with gadgets to both write **and** read arbitrary properties. But in constructor pollution the reading gadget must read from `constructor.polluted` instead of `polluted`. This dramatically reduces the number of useful gadgets.

### Computed access in minimized JS

Some minimized JS may be incompatible with encapsulation mode, because static property access could be minified into computed access. We have queried the [HTTP Archive](http://httparchive.org) to get an estimate of this in practice. The following table shows that pages with this behavior are consistently below 1% throughout the last 12 months for all pages crawled with a desktop browser:

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

### Example vulnerabilities

Google has seen an upward trend in bugs submitted to our Vulnerability Rewards Program: 1 in 2020, 3 in 2021 and 5 so far in 2022. We have identified several more in our internal research.

Example vulnerabilities include:

1. **On the Web**: Several XSS issues in services that should have been protected because they use [Strict CSP](https://w3c.github.io/webappsec-csp/#strict-csp). And a [wide range of known vulnerable libraries](https://github.com/BlackFan/client-side-prototype-pollution).
1. **On the desktop**: An bug in a Google-owned desktop application where users could be given a malicious JSON object that could allow local files to be leaked due to a pollution vulnerability. (Currently non-public, disclosure TBD.)
1. **In security features**: Multiple bypasses in sanitizers, including [Chrome's Sanitizer API](https://crbug.com/1306450), [DOMPurify and the Closure sanitizer](https://research.securitum.com/prototype-pollution-and-bypassing-client-side-html-sanitizers/).
1. **In the browser**: A [Firefox sandbox escape](https://www.zerodayinitiative.com/blog/2022/8/23/but-you-told-me-you-were-safe-attacking-the-mozilla-firefox-renderer-part-2) leading to remote code execution.
1. **In NodeJS**: Several [RCE](https://research.securitum.com/prototype-pollution-rce-kibana-cve-2019-7609/)s have been discovered.

We expect the number of vulnerable applications will grow as JavaScript applications are deployed to more environments (e.g. Electron, Cloudflare Workers, etc). Therefore, a language-level solution is required to mitigate attacks in all environments.


