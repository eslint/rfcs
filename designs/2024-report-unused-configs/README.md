- Repo: eslint/eslint
- Start Date: 2024-07-08
- RFC PR: https://github.com/eslint/rfcs/pull/121
- Authors: [Josh Goldberg](https://github.com/JoshuaKGoldberg)

# Reporting inline configs

## Summary

Add an option to report config file `rules` values and inline `/* eslint ... */` config comments that don't change any settings.

## Motivation

Right now, nothing in ESLint core stops [config files `rule` entries](https://eslint.org/docs/latest/use/configure/configuration-files#configuring-rules) or [inline configs (configuration comments)](https://eslint.org/docs/latest/use/configure/rules#using-configuration-comments) from redundantly repeating the same configuration _option_ and/or _severity_ as the existing computed configuration.
Both forms of configuration are undesirable when redundant: they take up space and can be misleading.

For example:

- [Stackblitz of an `eslint.config.js` setting a rule twice with the same severity](https://stackblitz.com/edit/stackblitz-starters-qcrxqr?file=eslint.config.js)
- [Stackblitz of an `eslint.config.js` setting a rule twice with the same options](https://stackblitz.com/edit/stackblitz-starters-san7bj?file=eslint.config.js)
- [Playground of an inline config setting the same severity as the config file](https://eslint.org/play/#eyJ0ZXh0IjoiLyogZXNsaW50IG5vLXVudXNlZC1leHByZXNzaW9uczogXCJlcnJvclwiICovXG5cIuKdjCBkb2VzIG5vdGhpbmc6IGV4aXN0aW5nIHNldmVyaXR5IHZzLiBmaWxlXCJcbiIsIm9wdGlvbnMiOnsicnVsZXMiOnsibm8tdW51c2VkLWV4cHJlc3Npb25zIjpbImVycm9yIl19LCJsYW5ndWFnZU9wdGlvbnMiOnsiZWNtYVZlcnNpb24iOiJsYXRlc3QiLCJzb3VyY2VUeXBlIjoibW9kdWxlIiwicGFyc2VyT3B0aW9ucyI6eyJlY21hRmVhdHVyZXMiOnt9fX19fQ==)
- [Playground of an inline config setting the same options as the config file](https://eslint.org/play/#eyJ0ZXh0IjoiLyogZXNsaW50IG5vLXVudXNlZC1leHByZXNzaW9uczogW1wiZXJyb3JcIiwgeyBcImFsbG93U2hvcnRDaXJjdWl0XCI6IHRydWUgfV0gKi9cblwi4p2MIGRvZXMgbm90aGluZzogZXhpc3Rpbmcgc2V2ZXJpdHkgYW5kIG9wdGlvbnMgdnMuIGZpbGVcIlxuIiwib3B0aW9ucyI6eyJydWxlcyI6eyJuby11bnVzZWQtZXhwcmVzc2lvbnMiOlsiZXJyb3IiLHsiYWxsb3dTaG9ydENpcmN1aXQiOnRydWUsImFsbG93VGVybmFyeSI6ZmFsc2UsImFsbG93VGFnZ2VkVGVtcGxhdGVzIjpmYWxzZSwiZW5mb3JjZUZvckpTWCI6ZmFsc2V9XX0sImxhbmd1YWdlT3B0aW9ucyI6eyJlY21hVmVyc2lvbiI6ImxhdGVzdCIsInNvdXJjZVR5cGUiOiJtb2R1bGUiLCJwYXJzZXJPcHRpb25zIjp7ImVjbWFGZWF0dXJlcyI6e319fX19)

This RFC proposes adding the ability for ESLint to report on both of those kinds of unused configs:

- `--report-unused-configs` CLI option
- `linterOptions.reportUnusedConfigs` configuration file option

```shell
npx eslint --report-unused-inline-configs error
```

```js
{
  linterOptions: {
    reportUnusedConfigs: "error",
  }
}
```

These new options would be similar to the existing [`--report-unused-disable-directives(-severity)`](https://eslint.org/docs/latest/use/command-line-interface#--report-unused-disable-directives) and [`linterOptions.reportUnusedDisableDirectives`](https://eslint.org/docs/latest/use/configure/configuration-files#reporting-unused-disable-directives) options.

### Examples

The following table uses [`accessor-pairs`](https://eslint.org/docs/latest/rules/accessor-pairs) as an example.
It assumes the `accessor-pairs` default options from [feat: add meta.defaultOptions](https://github.com/eslint/eslint/pull/17656) of:

```json
{
  "enforceForClassMembers": true,
  "getWithoutSet": false,
  "setWithoutGet": true
}
```

Values apply both for config files like `rules: { "accessor-pairs": "off" }` and inline configs like `/* eslint accessor-pairs: "off" */`.

<table>
  <thead>
    <tr>
      <th>First Settings</th>
      <th>Second Settings</th>
      <th>Report?</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="7">
        <code>"error"</code>
        <br />
        <code>2</code>
        <br />
        <code>["error", { "enforceForClassMembers": true }]</code>
        <br />
        <code>["error", { "getWithoutSet": false }]</code>
      </td>
      <td>
        <code>"off"</code>
        <br />
        <code>0</code>
      </td>
      <td>No</td>
    </tr>
    <tr>
      <td>
        <code>"warn"</code>
        <br />
        <code>1</code>
      </td>
      <td>No</td>
    </tr>
    <tr>
      <td>
        <code>"error"</code>
        <br />
        <code>2</code>
      </td>
      <td><strong>Yes</strong></td>
    </tr>
    <tr>
      <td>
        <code>["error", { "enforceForClassMembers": false }]</code>
        <br />
        <code>[2, { "enforceForClassMembers": false }]</code>
      </td>
      <td>No</td>
    </tr>
    <tr>
      <td>
        <code>["error", { "getWithoutSet": true }]</code>
        <br />
        <code>[2, { "getWithoutSet": true }]</code>
      </td>
      <td>No</td>
    </tr>
    <tr>
      <td>
        <code>["error", { "enforceForClassMembers": true }]</code>
        <br />
        <code>[2, { "enforceForClassMembers": true }]</code>
      </td>
      <td><strong>Yes</strong></td>
    </tr>
      <td>
        <code>["error", { "getWithoutSet": false }]</code>
        <br />
        <code>[2, { "getWithoutSet": false }]</code>
      </td>
      <td><strong>Yes</strong></td>
    </tr>
  </tbody>
</table>

## Detailed Design

This RFC proposes changing logical entry points only for the current ("flat") config system.
Legacy ("eslintrc") configs are omitted to reduce maintenance costs.

This proposed design does intentionally not involve any language-specific code changes.
How a specific language computes its configuration comments is irrelevant to this proposed feature.

### Glossary

For the sake of specificity, the following terms are used intentionally:

- **Options**: The options passed to a rule, as described by its `meta.schema`
- **Setting**: A configuration entry specifying a rule name, its severity, and optionally its _options_
- **Config file entry**: In an `eslint.config.js` file `rules` object, a key-value pair representing a single rule's _settings_
- **Inline config**: In a source file, a comment representing a single rule's _settings_

### Logic Entry Points

For rough code references, see:

- Config file entries: _(todo)_
- Inline config comments: [`poc: reporting unused inline configs`](https://github.com/JoshuaKGoldberg/eslint/commit/e14e404ed93e6238bdee817923a449f5215eecd8)


### Computing Option Differences

Each setting will be compared against the existing configuration value(s) it attempts to override:

- If the setting only specifies a severity, then only the severity will be checked for redundancy
  - The new logic will normalize options: `"off"` will be considered equivalent to `0`
- If the setting also specifies rule options, they will be compared for deep equality to the existing rule options

This RFC should wait to begin work until after [feat: add meta.defaultOptions](https://github.com/eslint/eslint/pull/17656) is merged into ESLint.
That way, a rule's `meta.defaultOptions` can be factored into computing whether an setting's rule options differ from the previously configured options.

### Config Files

Only configs in the user's code will be checked.
Third-party shareable configs will not be validated.

For config files, that means only `rules` entries in the user's specified config file will be checked.

### Default Values

This RFC proposes a two-step approach to introducing unused config reporting:

1. In the current major version upon introduction, don't enable unused config reporting by default
2. In the next major version, enable unused config reporting by default

Note that the default value in the next major version should be the same as reporting unused disable directives, `"warn"`.

## Documentation

The new settings will be documented similarly to reporting unused disable directives:

- [Configuration Files](https://eslint.org/docs/latest/use/configure/configuration-files):
  - List item for `reportUnusedInlineConfig` under _[Configuration Objects](https://eslint.org/docs/latest/use/configure/configuration-files#configuration-objects)_ > `linterOptions`
  - Sub-heading alongside _[Reporting Unused Disable Directives](https://eslint.org/docs/latest/use/configure/configuration-files#reporting-unused-disable-directives)_
- [Configure Rules](https://eslint.org/docs/latest/use/configure/rules):
  - Sub-heading alongside _[Report unused eslint-disable comments](https://eslint.org/docs/latest/use/configure/rules#report-unused-eslint-disable-comments)_
- [Command Line Interface Reference](https://eslint.org/docs/latest/use/command-line-interface):
  - List item under the _[Options](https://eslint.org/docs/latest/use/command-line-interface#options)_ code block, under `Inline Configuration comments:`
  - Corresponding sub-headings under _[Inline Configuration Comments](https://eslint.org/docs/latest/use/command-line-interface#inline-configuration-comments)_

## Drawbacks

Any added options come with an added tax on project maintenance and user comprehension.
This RFC believes the flagging of unused inline configs is worth that tax.

See [Omitting Legacy Config Support](#omitting-legacy-config-support) for a possible reduction in cost.

## Backwards Compatibility Analysis

The proposed two-step approach introduces the options in a fully backwards-compatible way.
No new warnings or errors will be reported in the current major version without the user explicitly opting into them.

## Alternatives

### Deferring to the Config Inspector

TODO

### Omitting Legacy Config Support

One way to reduce costs could be to wait until ESLint completely removes support for legacy configs.
That way, only the new ("flat") config system would need to be tested with this change.

However, it is unclear when the legacy config system will be removed from ESLint core.
This RFC prefers taking action sooner rather than later.

### Separate CLI Option

Unlike the changes discussed in [Change Request: Enable reportUnusedDisableDirectives config option by default](https://github.com/eslint/eslint/issues/15466) -> [feat!: flexible config + default reporting of unused disable directives](https://github.com/eslint/rfcs/pull/100), reporting unused inline configs does not have legacy behavior to keep to.
The existing `--report-unused-disable-directives` (enabling) and `--report-unused-disable-directives-severity` (severity) options were kept separate for backwards compatibility.

Adding a sole `--report-unused-inline-configs` CLI option presents a discrepency between the two sets of options.
An alternative could be to instead add `--report-unused-inline-configs` and `--report-unused-inline-configs-severity` options for consistency's sake.

This RFC's opinion is that the consistency of adding two new options is not worth the excess options logic.
It would instead be preferable to, in a future ESLint major version, deprecate `--report-unused-disable-directives-severity` and merge its logic setting into `--report-unused-disable-directives`.

### Superset Behavior: Unused Disable Directive Reporting

Disable directives can be thought as a subset of inline configs in general.
Reporting on unused disable directives could be thought of as a subset of reporting on unused inline configs.

An additional direction this RFC could propose would be to have the new unused inline config reporting act as a superset of unused disable directive reporting.

However:

- Deprecating `reportUnusedDisableDirectives` would be a distruptive breaking change
- This RFC proposes enabling `reportUnusedConfigs` by default in a subsequent major version anyway: most users will not be manually configuring it

## Help Needed

I would like to implement this RFC.

## Frequently Asked Questions

### Why so many references to reporting unused disable directives?

This RFC's proposed behavior is similar on the surface to the existing behavior around reporting unused disable directives.
It's beneficial for users to have similar behaviors between similar options.

### Will inline configs be compared to previous inline configs in the same file?

As of [Change Request: Disallow multiple configuration comments for the same rule](https://github.com/eslint/eslint/issues/18132) -> [feat!: disallow multiple configuration comments for same rule](https://github.com/eslint/eslint/pull/18157), the same rule cannot be configured by an inline config more than once per file.

## Related Discussions

- <https://github.com/eslint/eslint/issues/15476>: issue suggesting to report on unused config file comments
- <https://github.com/eslint/eslint/issues/18230>: issue suggesting to report on unused inline config comments
- <https://github.com/eslint/eslint/issues/15466>: previous issue for enabling `reportUnusedDisableDirectives` config option by default
  - <https://github.com/eslint/rfcs/pull/100>: the RFC discussion flexible config + default reporting of unused disable directives
  - <https://github.com/eslint/eslint/pull/17212>: the PR implementing custom severity when reporting unused disable directives
- <https://github.com/eslint/eslint/issues/18665>: issue suggesting erroring by default for unused disable directives
- <https://github.com/eslint/eslint/issues/18666>: issue suggesting merging `--report-unused-disable-directives-severity` into `--report-unused-disable-directives`
