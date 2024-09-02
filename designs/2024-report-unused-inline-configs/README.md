- Repo: eslint/eslint
- Start Date: 2024-07-08
- RFC PR: https://github.com/eslint/rfcs/pull/121
- Authors: [Josh Goldberg](https://github.com/JoshuaKGoldberg)

# Reporting unused inline configs

## Summary

Add an option to report `/* eslint ... */` comments that don't change any settings.

## Motivation

Right now, nothing in ESLint core stops [inline configs (configuration comments)](https://eslint.org/docs/latest/use/configure/rules#using-configuration-comments) from redundantly repeating the same configuration _option_ and/or _severity_ as the file's existing computed configuration.
Unused inline configs suffer from the same drawbacks as [unused disable directives](https://eslint.org/docs/latest/use/configure/rules#report-unused-eslint-disable-comments): they take up space and can be misleading.

For example:

- [Playground of an inline config setting the same severity as the config file](https://eslint.org/play/#eyJ0ZXh0IjoiLyogZXNsaW50IG5vLXVudXNlZC1leHByZXNzaW9uczogXCJlcnJvclwiICovXG5cIuKdjCBkb2VzIG5vdGhpbmc6IGV4aXN0aW5nIHNldmVyaXR5IHZzLiBmaWxlXCJcbiIsIm9wdGlvbnMiOnsicnVsZXMiOnt9LCJsYW5ndWFnZU9wdGlvbnMiOnsiZWNtYVZlcnNpb24iOiJsYXRlc3QiLCJzb3VyY2VUeXBlIjoibW9kdWxlIiwicGFyc2VyT3B0aW9ucyI6eyJlY21hRmVhdHVyZXMiOnt9fX19fQ==)
- [Playground of an inline config setting the same options as the config file](https://eslint.org/play/#eyJ0ZXh0IjoiLyogZXNsaW50IG5vLXVudXNlZC1leHByZXNzaW9uczogW1wiZXJyb3JcIiwgeyBcImFsbG93U2hvcnRDaXJjdWl0XCI6IHRydWUgfV0gKi9cblwi4p2MIGRvZXMgbm90aGluZzogZXhpc3Rpbmcgc2V2ZXJpdHkgYW5kIG9wdGlvbnMgdnMuIGZpbGVcIlxuIiwib3B0aW9ucyI6eyJydWxlcyI6eyJuby11bnVzZWQtZXhwcmVzc2lvbnMiOlsiZXJyb3IiLHsiYWxsb3dTaG9ydENpcmN1aXQiOnRydWUsImFsbG93VGVybmFyeSI6ZmFsc2UsImFsbG93VGFnZ2VkVGVtcGxhdGVzIjpmYWxzZSwiZW5mb3JjZUZvckpTWCI6ZmFsc2V9XX0sImxhbmd1YWdlT3B0aW9ucyI6eyJlY21hVmVyc2lvbiI6ImxhdGVzdCIsInNvdXJjZVR5cGUiOiJtb2R1bGUiLCJwYXJzZXJPcHRpb25zIjp7ImVjbWFGZWF0dXJlcyI6e319fX19)

This RFC proposes adding the ability for ESLint to report on those unused inline configs:

- `--report-unused-inline-configs` CLI option
- `linterOptions.reportUnusedInlineConfigs` configuration file option

```shell
npx eslint --report-unused-inline-configs error
```

```js
{
  linterOptions: {
    reportUnusedInlineConfigs: "error",
  }
}
```

These new options would be similar to the existing [`--report-unused-disable-directives(-severity)`](https://eslint.org/docs/latest/use/command-line-interface#--report-unused-disable-directives) and [`linterOptions.reportUnusedDisableDirectives`](https://eslint.org/docs/latest/use/configure/configuration-files#reporting-unused-disable-directives) options.
However, this RFC proposes a single option to both enable the report and configure its severity, rather than two separate options.

### Examples

The following table uses [`accessor-pairs`](https://eslint.org/docs/latest/rules/accessor-pairs) as an example with inline configs like `/* eslint accessor-pairs: "off" */`.
It assumes the `accessor-pairs` default options from [feat: add meta.defaultOptions](https://github.com/eslint/eslint/pull/17656) of:

```json
{
  "enforceForClassMembers": true,
  "getWithoutSet": false,
  "setWithoutGet": true
}
```

<table>
  <thead>
    <tr>
      <th>Original Config Setting</th>
      <th>Inline Config Settings</th>
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

Additional logic can be added to the existing code points in `Linter` that validate inline config options: [`_verifyWithFlatConfigArrayAndWithoutProcessors`](https://github.com/eslint/eslint/blob/7c78ad9d9f896354d557f24e2d37710cf79a27bf/lib/linter/linter.js#L1636).
For a rough code reference, see [`poc: reporting unused inline configs`](https://github.com/JoshuaKGoldberg/eslint/commit/e14e404ed93e6238bdee817923a449f5215eecd8).

### Computing Option Differences

Each inline config comment will be compared against the existing configuration value(s) it attempts to override:

- If the config comment only specifies a severity, then only the severity will be checked for redundancy
  - The new logic will normalize options: `"off"` will be considered equivalent to `0`
- If the config comment also specifies rule options, they will be compared for deep equality to the existing rule options

There will be a report if both the severity and rule options are the same.
If there is a difference in the severity and/or rule options, then there will be no report.

This RFC should wait to begin work until after [feat: add meta.defaultOptions](https://github.com/eslint/eslint/pull/17656) is merged into ESLint.
That way, if a rule defines `meta.defaultOptions`, those default options can be factored into computing whether an inline config's rule options differ from the previously configured options.

### Default Values

This RFC proposes a two-step approach to introducing unused inline config reporting:

1. In the current major version upon introduction, don't enable unused inline config reporting by default
2. In the next major version, enable unused inline config reporting by default

Note that the default value in the next major version should be the same as reporting unused disable directives.
See [Change Request: error by default for unused disable directives](https://github.com/eslint/eslint/issues/18665) for an issue on changing that from `"warn"` to `"error"`.

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

## Backwards Compatibility Analysis

The proposed two-step approach introduces the options in a fully backwards-compatible way.
No new warnings or errors will be reported in the current major version without the user explicitly opting into them.

## Performance Analysis

This RFC believes there will be no nontrivial performance impact from this change.
All rule lookups from inline configs are O(1) compared to the existing computed rules for a file.
It's rare in userland code to have more than a single digit number of inline configs in any file.

## Out of Scope

### Language-Specific Changes

This proposed design does intentionally not involve any language-specific code changes.
How a specific language computes its configuration comments is irrelevant to this proposed feature.

### Legacy ("eslintrc") Configs

ESLint v9 is released and flat configs are the default for users.
The legacy config system will likely be removed in the next major version of ESLint.
This RFC prefers to avoid the added maintenance cost of supporting the legacy config system.

If this RFC's new CLI flag or config file entry are enabled with using a legacy config, ESLint should throw an error.

## Alternatives

### Checking Config File Values

[eslint/eslint#15476 Change Request: report unnecessary config overrides](https://github.com/eslint/eslint/issues/15476) previously suggested also checking config files (`eslint.config.*`).
Doing so could be beneficial to flag values that become unnecessary in config files over time.
However, because flat config purely involves spreading objects, there's no way to know what objects originate from shared configs or shared packages.

[c339a5](https://github.com/eslint/rfcs/pull/121/commits/c339a59bfaec0c48817a012c2f5e92a242a1b1e6) is a reference commit of how this RFC might look with that support added in.

Instead, this RFC suggests that the [config inspector](https://github.com/eslint/config-inspector) would be a more natural place to see these conflicts.

An alternative could have been to require users provide metadata along with their shared configs, and/or wrap them in some function provided by ESLint.
That could be a future requirement opted into ESLint.
It's too late now to add that to the flat config system, and as such is out of scope for this RFC.

### Separate CLI Option

Unlike the changes discussed in [Change Request: Enable reportUnusedDisableDirectives config option by default](https://github.com/eslint/eslint/issues/15466) -> [feat!: flexible config + default reporting of unused disable directives](https://github.com/eslint/rfcs/pull/100), reporting unused inline configs does not have legacy behavior to keep to.
The existing `--report-unused-disable-directives` (enabling) and `--report-unused-disable-directives-severity` (severity) options were kept separate for backwards compatibility.

Adding a sole `--report-unused-inline-configs` CLI option presents a discrepency between the two sets of options.
An alternative could be to instead add `--report-unused-inline-configs` and `--report-unused-inline-configs-severity` options for consistency's sake.

This RFC's opinion is that the consistency of adding two new options is not worth the excess options logic.

### Superset Behavior: Unused Disable Directive Reporting

Disable directives can be thought as a subset of inline configs in general.
Reporting on unused disable directives could be thought of as a subset of reporting on unused inline configs.

An additional direction this RFC could propose would be to have the new unused inline config reporting act as a superset of unused disable directive reporting.

However, deprecating `reportUnusedDisableDirectives` would be a disruptive change and eventual disruptive breaking change.
This RFC prefers keeping away from larger changes like that.
A future change in a subsequent major version could take that on separately.

## Help Needed

I would like to implement this RFC.

## Frequently Asked Questions

### Why so many references to reporting unused disable directives?

This RFC's proposed behavior is similar on the surface to the existing behavior around reporting unused disable directives.
It's beneficial for users to have similar behaviors between similar options.

### Will inline configs be compared to previous inline configs in the same file?

As of [Change Request: Disallow multiple configuration comments for the same rule](https://github.com/eslint/eslint/issues/18132) -> [feat!: disallow multiple configuration comments for same rule](https://github.com/eslint/eslint/pull/18157), the same rule cannot be configured by an inline config more than once per file.

## Related Discussions

- <https://github.com/eslint/eslint/issues/18230>: the issue triggering this RFC
  - <https://github.com/eslint/eslint/issues/15476>: previous issue suggesting reporting unnecessary config overrides
- <https://github.com/eslint/eslint/issues/15466>: previous issue for enabling `reportUnusedDisableDirectives` config option by default
  - <https://github.com/eslint/rfcs/pull/100>: the RFC discussion flexible config + default reporting of unused disable directives
  - <https://github.com/eslint/eslint/pull/17212>: the PR implementing custom severity when reporting unused disable directives
- <https://github.com/eslint/eslint/issues/18665>: issue suggesting erroring by default for unused disable directives
- <https://github.com/eslint/eslint/issues/18666>: issue suggesting merging `--report-unused-disable-directives-severity` into `--report-unused-disable-directives`
