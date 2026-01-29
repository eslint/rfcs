- Repo: <https://github.com/eslint/eslint>
- Start Date: 2024-10-22
- RFC PR: Leave blank until PR is created
- Authors:
  - [Samuel Therrien](https://github.com/Samuel-Therrien-Beslogic) (aka [@Avasam](https://github.com/Avasam))
  - [Maxim Morev](https://github.com/MorevM)
  - [Amaresh S M](https://www.linkedin.com/in/amaresh-s-m/)

# Per-Rule Autofix Control via Object-Based Rule Configuration

## Summary

This RFC proposes extending ESLint's rule configuration to support an optional object format, allowing users to disable autofixes on a per-rule basis.

The key point is that we can add this feature without affecting anything that already works — the current "error" string and ["error", options] array formats will keep working exactly the same.

This proposal builds on the consensus reached in [RFC #125](https://github.com/eslint/rfcs/pull/125), where the team agreed that an object-based approach provides the best balance of flexibility and backward compatibility.

## Motivation

### Problem

As discussed in [issue #18696](https://github.com/eslint/eslint/issues/18696), there's a real need for per-rule autofix control. While ESLint's autofix feature is incredibly useful, there are legitimate scenarios where you want a rule enabled (to catch issues) but don't want its autofix applied automatically:

1. **Unsafe autofixes**: Some rules have autofixes that might change code behavior in edge cases. You want to catch these issues but review fixes manually.

2. **Unmaintained plugins**: When using plugins that are no longer actively maintained, you might encounter broken autofixes. Disabling the entire rule means losing valuable linting, but applying broken fixes is worse.

3. **Manual review requirements**: Some teams have policies requiring human review of certain types of changes, even if technically "safe" to autofix.

4. **CI/CD workflows**: You might want different behavior in CI (report only) vs local development (autofix enabled).

### Available workarounds:

Right now, users work around this limitation using [`eslint-plugin-no-autofix`](https://www.npmjs.com/package/eslint-plugin-no-autofix). While this plugin has been helpful, it comes with real downsides:

- **Extra dependency**: Another package to maintain and keep updated
- **Environment issues**: Doesn't work in all environments (e.g., [pre-commit.ci crashes](https://github.com/aladdin-add/eslint-plugin/issues/98))
- **Compatibility problems**: May not work correctly with all third-party rules (see [eslint-community/eslint-plugin-eslint-comments#234](https://github.com/eslint-community/eslint-plugin-eslint-comments/issues/234))
- **Cumbersome setup**: Rules must use a no-autofix/ prefix, and autofix control is separate from the rule itself

A native solution would eliminate these issues entirely.

### Real-World Use Cases

Here are some concrete scenarios where this feature would be valuable:

1. **Disable unsafe autofixes**: The `no-implicit-coercion` rule is great for catching issues, but its autofix can occasionally change behavior in subtle ways. You want the warnings, but prefer to review and apply fixes manually.
   ```javascript
   rules: {
     "no-implicit-coercion": { severity: "error", autofix: false }
   }
   ```

2. **Preserve imports during development**: As described in [issue #18696](https://github.com/eslint/eslint/issues/18696#issuecomment-2429108285), teams using `eslint-plugin-unused-imports` face a frustrating workflow issue - when you comment out code during development, autofixes immediately remove the imports. When you uncomment the code, you have to manually re-add all the imports. The solution is to keep the rule enabled (to catch actual unused imports) but disable its autofix.
   ```javascript
   rules: {
     "unused-imports/no-unused-imports": { severity: "error", autofix: false }
   }
   ```

3. **Override extended configs**: As mentioned in [RFC #125](https://github.com/eslint/rfcs/pull/125#issuecomment-2437891333), you might want `@eslint-community/eslint-comments/no-unused-disable` to report issues but not autofix them, since a rule suddenly not failing should require human attention.
   ```javascript
   import recommended from 'eslint-config-recommended';
   export default [
     recommended,
     {
       rules: {
         "@eslint-community/eslint-comments/no-unused-disable": { autofix: false }
       }
     }
   ];
   ```

4. **Environment-specific behavior**: Different autofix settings for CI vs local development .
   ```javascript
   const isCI = process.env.CI === 'true';
   export default [{
     rules: {
       "semi": { severity: "error", autofix: !isCI }
     }
   }];
   ```

## Detailed Design

### Rule Configuration Format

The core idea is simple: extend ESLint's rule configuration to accept an **optional object format** alongside the current string and array formats. This is just adition - nothing breaks, nothing changes unless you opt in.

As discussed in [RFC #125](https://github.com/eslint/rfcs/pull/125#discussion_r1929041731), this approach was chosen over alternatives (like a top-level `nofix` key) because it keeps all rule settings in one place and allows for additional options for each rule (as discussed in [#19342](https://github.com/eslint/eslint/issues/19342)).

```javascript
export default [{
  rules: {
    // Existing formats (continue to work)
    "semi": "error",  // String format
    "semi": ["error"], // Array format
    "quotes": ["error", "double"], // Array format

    // New object format (opt-in)
    "no-unused-vars": {
      severity: "error",
      options: [{ "vars": "all" }],
      autofix: false // New property
    }
  }
}];
```

### Object Properties

The object format supports the same properties you're already familiar with from array format, plus a new one:

- **`severity`**: Same as the first element in array format (`"off"`, `"warn"`, `"error"`, `0`, `1`, or `2`)
- **`options`**: Same as the remaining elements in array format (rule-specific options)
- **`autofix`** *(new)*: Boolean to control whether autofixes are applied for this rule (default: `true`)

### Partial Object Merging

This is where things get really useful. As [@nzakas pointed out in RFC #125](https://github.com/eslint/rfcs/pull/125#issuecomment-2582026498), we can support **partial object merging** - you only need to specify the properties you want to change:

> "I think we can stipulate that when merging, we'll just merge the keys that are specified and keep everything else."

This means you don't need to know or repeat the full rule configuration just to disable autofix:

```javascript
// base.config.js
export default [{
  rules: {
    "semi": {
      severity: "error",
      options: [],
      autofix: true
    }
  }
}];

// extended.config.js
import base from './base.config.js';
export default [
  base,
  {
    rules: {
      "semi": { autofix: false }  // Only changes autofix, inherits severity & options
    }
  }
];

// Result: { severity: "error", options: [], autofix: false }
```

Merge priority: Later configs override earlier ones. In the example above:
- `base.config.js` sets `autofix: true`
- `extended.config.js` sets `autofix: false`
- **Result**: `autofix: false` wins (later config takes priority)
- Properties not specified in the later config (`severity`, `options`) are inherited from the earlier config

### File-Specific Overrides

One of the concerns raised in [RFC #125](https://github.com/eslint/rfcs/pull/125#issuecomment-2437891333) was the ability to re-enable autofixes in specific contexts. The object format handles this naturally through ESLint's existing file-based configuration:

```javascript
export default [
  {
    rules: {
      "semi": {
        severity: "error",
        autofix: false  // Disabled globally
      }
    }
  },
  {
    files: ["**/*.test.js"],
    rules: {
      "semi": { autofix: true }  // Re-enabled for test files
    }
  }
];
```

### CLI and Inline Comment Support

As [@fasttime noted in RFC #125](https://github.com/eslint/rfcs/pull/125#discussion_r1835013306), it's important that rules can be configured consistently across different methods - config files, CLI, and inline comments. The beauty of the object-based approach is that it works everywhere with **no new syntax**:

**CLI** (existing `--rule` syntax):
```bash
eslint --rule 'semi: { "autofix": false }' src/
```

**Inline comments** (existing `/* eslint */` syntax):
```javascript
/* eslint semi: { "autofix": false } */
function foo() {
  return 1
}
```

### Implementation Details

#### Step 1: Config Schema (`lib/config/flat-config-schema.js`)

We need to update the `rules` schema to handle object-based configurations. The key is the `normalizeRuleConfig()` helper that converts all formats to a common internal representation:

```javascript
rules: {
  merge(first = {}, second = {}) {
    const result = { ...first };

    for (const [ruleId, ruleConfig] of Object.entries(second)) {
      const firstConfig = first[ruleId];
      const secondConfig = ruleConfig;

      // Normalize both configs to objects for merging
      const firstObj = normalizeRuleConfig(firstConfig);
      const secondObj = normalizeRuleConfig(secondConfig);

      // Merge: second overrides first (partial merge)
      result[ruleId] = { ...firstObj, ...secondObj };
    }

    return result;
  },

  validate(value) {
    // Existing validation for string/array formats
    // ... existing code ...

    // Add validation for object format
    // Why we need this: Currently, ESLint's validation (lib/config/flat-config-schema.js:184-192)
    // REJECTS objects via assertIsRuleOptions(), which only allows string/number/array.
    // The object format is a completely new code path that needs its own validation.
    for (const [ruleId, ruleConfig] of Object.entries(value)) {
      if (typeof ruleConfig === 'object' && !Array.isArray(ruleConfig)) {
        // Validate object format
        const validKeys = ['severity', 'options', 'autofix'];
        const configKeys = Object.keys(ruleConfig);

        for (const key of configKeys) {
          if (!validKeys.includes(key)) {
            throw new TypeError(
              `Rule "${ruleId}": Unknown property "${key}". Valid properties: ${validKeys.join(', ')}`
            );
          }
        }

        // Validate severity (same validation as array format, but for object property)
        if ('severity' in ruleConfig) {
          validateSeverity(ruleConfig.severity);
        }

        // Validate options (must be array, same as array format's remaining elements)
        if ('options' in ruleConfig) {
          if (!Array.isArray(ruleConfig.options)) {
            throw new TypeError(`Rule "${ruleId}": 'options' must be an array`);
          }
        }

        // Validate autofix (NEW property introduced by this RFC)
        if ('autofix' in ruleConfig) {
          if (typeof ruleConfig.autofix !== 'boolean') {
            throw new TypeError(`Rule "${ruleId}": 'autofix' must be a boolean`);
          }
        }
      }
    }
  }
}

// Helper function to normalize rule configs
// This is the key to maintaining backward compatibility - we convert
// all formats (string, array, object) to a common internal representation
function normalizeRuleConfig(config) {
  // Handle edge case: null/undefined configs
  if (config === null || config === undefined) {
    return {};
  }

  // String format: "error" -> { severity: "error" }
  if (typeof config === 'string' || typeof config === 'number') {
    return { severity: config };
  }

  // Array format: ["error", options] -> { severity: "error", options: [options] }
  if (Array.isArray(config)) {
    return {
      severity: config[0],
      options: config.slice(1)
    };
  }

  // Object format: already normalized
  if (typeof config === 'object') {
    return config;
  }

  return {};
}
```

#### Example transformations:

```javascript
// Input: String format
rules: { "semi": "error" }
// After normalization:
{ "semi": { severity: "error" } }

// Input: Array format
rules: { "quotes": ["error", "double"] }
// After normalization:
{ "quotes": { severity: "error", options: ["double"] } }

// Input: Array format (severity only)
rules: { "no-console": ["warn"] }
// After normalization:
{ "no-console": { severity: "warn", options: [] } }

// Input: Object format (new)
rules: { "semi": { severity: "error", autofix: false } }
// After normalization:
{ "semi": { severity: "error", autofix: false } }

// Input: Object format (partial - only autofix)
rules: { "semi": { autofix: false } }
// After normalization (when merged with base config):
{ "semi": { severity: "error", options: [], autofix: false } }
// (severity and options inherited from earlier config)
```

#### Step 2: Linter Integration (`lib/linter/linter.js`)

The linter needs to check the `autofix` property and suppress fixes when it's `false`. The important part here is that we **only suppress fixes, not suggestions** - this was clarified in the discussion, as suggestions require explicit user action and shouldn't be affected by autofix settings.

```javascript
// In runRules function, when setting up rule context:
Object.keys(configuredRules).forEach(ruleId => {
  const ruleConfig = configuredRules[ruleId];

  // Normalize to object format to access autofix property
  const normalizedConfig = normalizeRuleConfig(ruleConfig);

  // NEW: Check if autofix is disabled for this rule
  const autofixDisabled = normalizedConfig.autofix === false;

  const ruleContext = fileContext.extend({
    id: ruleId,
    options: normalizedConfig.options || [],
    report(...args) {
      const problem = report.addRuleMessage(ruleId, severity, ...args);

      // NEW: Suppress fix if autofix is disabled for this rule
      if (autofixDisabled && problem.fix) {
        problem.fix = null;
      }

      // Suggestions remain enabled - they require explicit user action
      // and shouldn't be affected by autofix settings
    },
  });

  // ... rest of rule execution
});
```

**What this does:**
- Reads the `autofix` property from the normalized config
- When a rule reports a problem with a fix, checks if `autofixDisabled` is true
- If so, nulls out the fix before the problem is recorded
- Leaves suggestions untouched so they remain available to the user

#### Step 3: Inline Config Comment Support

Inline config comments (like `/* eslint semi: { "autofix": false } */`) will work automatically once Steps 1 and 2 are implemented. Here's why:

- Inline config parsing happens in `SourceCode.applyInlineConfig()` (`lib/languages/js/source-code/source-code.js:981`)
- It uses `ConfigCommentParser.parseJSONLikeConfig()` from `@eslint/plugin-kit` to parse the comment value
- The parsed config is added to the `rules` object, which then goes through the same validation and normalization as file-based configs
- Since we're updating the `rulesSchema` in Step 1, inline configs will automatically support the object format

**No additional code changes needed** - the existing inline config infrastructure will handle object format once the schema is updated.

### Behavior Specifications

1. **Default value**: When `autofix` is not specified, it defaults to `true` (maintains current behavior - no changes for existing configs)

2. **Suggestions preserved**: When `autofix: false`, suggestions remain available. This is intentional - suggestions require explicit user action

3. **Rule still reports**: Setting `autofix: false` does NOT disable the rule. Violations are still detected and reported. This is the whole point - you want the linting, just not the automatic fixing.

4. **Merging behavior**: Later configs override only the properties they specify. If you specify `{ autofix: false }`, it inherits `severity` and `options` from earlier configs.

5. **CLI precedence**: CLI `--rule` flags override config file settings (this is existing ESLint behavior, unchanged).

## Documentation

### User Documentation Updates

1. **Configuration Guide** (`docs/src/use/configure/rules.md`):
   - Add new section "Rule Configuration Formats" in the "Using Configuration Files" section (after line 113)
   - Document the three formats: string, array, and object
   - Explain the new `autofix` property and its behavior
   - Show partial object merging with examples
   - Provide use cases for disabling autofix while keeping rule enabled

2. **Node.js API Reference** (`docs/src/integrate/nodejs-api.md`):
   - Update the rule configuration documentation to mention the `autofix` property
   - Note that it's available in all config sources (files, CLI, inline comments)
   - Document default value is `true` (maintains backward compatibility)

### Example Documentation

Rule Configuration Formats - ESLint supports three formats for configuring rules:

#### String Format
```javascript
rules: {
  "semi": "error"
}
```

#### Array Format
```javascript
rules: {
  "quotes": ["error", "double"]
}
```

#### Object Format
```javascript
rules: {
  "no-unused-vars": {
    severity: "error",
    options: [{ "vars": "all" }],
    autofix: false
  }
}
```

#### Disabling Autofixes

To disable autofixes for a specific rule while keeping it enabled:

```javascript
rules: {
  "semi": { autofix: false }  // Inherits severity from extended config
}
```

## Drawbacks

**No significant drawbacks identified.**

This proposal adds an optional configuration format. Users who don't need per-rule autofix control will never encounter it - their existing string and array configurations continue to work unchanged. Users who do need this functionality will find it intuitive since it follows the same patterns as existing ESLint configuration.

## Backwards Compatibility Analysis

This proposal is **100% backward compatible**

### No Breaking Changes

1. **Existing string format**: `"error"` continues to work
2. **Existing array format**: `["error", options]` continues to work
3. **No forced migration**: Object format is opt-in only
4. **All formats coexist**: Can mix string, array, and object formats in same config
5. **Default behavior unchanged**: When `autofix` is not specified, autofixes work as they do today

### Migration Path

Users can adopt the new format gradually:

```javascript
// Before (using eslint-plugin-no-autofix)
import noAutofix from 'eslint-plugin-no-autofix';

export default [{
  plugins: { "no-autofix": noAutofix },
  rules: {
    "no-autofix/semi": "error"
  }
}];
// After (native support)
export default [{
  rules: {
    "semi": { severity: "error", autofix: false }
  }
}];

// Or with partial object format
import baseConfig from './base.config.js';
export default [
  baseConfig,
  {
    rules: {
      "semi": { autofix: false }  // Inherits severity from base
    }
  }
];
```

## Alternatives

### Top-Level `nofix` Configuration

We could add a separate top-level `nofix` key to disable autofixes:

```javascript
export default [{
  rules: { "semi": "error" },
  nofix: ["semi"]
}];
```

This was actually my initial approach, Ref from [Ruff's `unfixable` setting](https://docs.astral.sh/ruff/settings/#lint_unfixable).

**Why we're not doing this**:
- Separates autofix control from rule configuration (less intuitive)
- Would require new CLI syntax (`--no-fix-rule` or similar)
- Would require new inline comment syntax
- Less discoverable - users would need to know about a separate config key
- Doesn't enable future per-rule meta options (like per-rule `reportUnusedDisableDirectives`)

The object-based approach keeps everything in one place and works with existing syntax everywhere.

## Help Needed

**Edge case identification**: This proposal done based on discussions from [RFC #125](https://github.com/eslint/rfcs/pull/125), [RFC #134](https://github.com/eslint/rfcs/pull/134), and related issues. I have addressed the scenarios raised in those discussions, but there may be edge cases or interaction patterns we haven't considered.

I might need help in identifying:
- Edge cases from multi-file config merging combined with complex configs/patterns
- Plugin-specific behaviors that might be affected

## Frequently Asked Questions

### Why not just disable the rule?

This is probably the most common question. The answer: disabling the rule (`"off"`) means violations won't be reported at all. You lose the linting entirely.

With `autofix: false`, violations are still reported (and suggestions still work), but the fix won't be applied automatically with `--fix`. You get the safety of the linting without the risk of automatic changes.

### What happens to suggestions?

Suggestions remain available even when `autofix: false`. This is intentional.

Suggestions require explicit user action (e.g., clicking in your IDE), so they're fundamentally different from automatic fixes. If you've disabled autofix because you want manual control, suggestions are exactly what you want - they give you the option to apply a fix with one click, but only when you choose to.

### Can I re-enable autofixes in overrides?

Yes! This was actually a key concern raised in [RFC #125](https://github.com/eslint/rfcs/pull/125#issuecomment-2437891333). The object format handles this naturally:

```javascript
export default [
  {
    rules: { "semi": { autofix: false } }  // Disabled globally
  },
  {
    files: ["**/*.test.js"],
    rules: { "semi": { autofix: true } }   // Enabled for tests
  }
];
```

### Does this work with shareable configs?

This is one of the main use cases - being able to use a shareable config but disable specific autofixes:

```javascript
import recommended from 'eslint-config-recommended';

export default [
  recommended,
  {
    rules: {
      "semi": { autofix: false }  // Override from recommended
    }
  }
];
```

### What's the default value for `autofix`?

When not specified, `autofix` defaults to `true` (current behavior). Autofixes are enabled by default.

### Can I use this with the `--fix` flag?

Yes. When you run `eslint --fix`, rules with `autofix: false` will report violations but won't apply fixes. Rules without the `autofix` property will apply fixes normally.

### How does this differ from `meta.fixable`?

- **`meta.fixable`**: Set by rule authors to indicate if a rule *can* provide fixes. This is part of the rule's definition.
- **`autofix`**: Set by users to control whether fixes *should* be applied in their project. This is part of the configuration.

## Related Discussions

This RFC builds on prior community discussions:

- **[eslint/eslint#18696](https://github.com/eslint/eslint/issues/18696)**: Original feature request for per-rule autofix control
- **[eslint/rfcs#125](https://github.com/eslint/rfcs/pull/125)**: RFC discussion where the object-based approach reached consensus
- **[eslint/rfcs#134](https://github.com/eslint/rfcs/pull/134)**: Alternative RFC (closed due to breaking changes)
- **[eslint/eslint#19342](https://github.com/eslint/eslint/issues/19342)**: Related discussion on extensible per-rule meta options
- **[eslint-community/eslint-plugin-eslint-comments#234](https://github.com/eslint-community/eslint-plugin-eslint-comments/issues/234)**: Related discussion on per-rule autofix control
