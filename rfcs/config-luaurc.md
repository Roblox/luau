# Configure analysis via .luaurc

## Summary

Introduces a way to configure type checker and linter using JSON-like .luaurc files

## Motivation

While Luau analysis tools try to provide sensible defaults, it's difficult to establish the rules that work for all code.
For example, some packages may decide that unused variables aren't interesting; other packages may decide that all files should be using strict typechecking mode.

While it's possible to configure some aspects of analysis behavior using --! comments, it can be cumbersome to replicate this in all files.

## Design

To solve this problem, we are going to introduce support for `.luaurc` files for users of command-line Luau tools.
For a given .lua file, Luau will search for .luaurc files starting from the folder that the .lua file is in; all files in the ancestry chain will be parsed and their configuration
applied. When multiple files are used, the file closer to the .lua file overrides the settings.

.luaurc is a JSON file that can also contain comments and trailing commas. The file can have the following keys:

- `"mode"`: type checking mode, can be one of "nocheck", "nonstrict", "strict"
- `"language"`: legacy specification of type checking mode, contains `"mode"` key - e.g. `{ "language": { "mode": "strict" } }`. Supported for backwards compatibility with existing .robloxrc files.
- `"lint"`: linting settings; points to an object that contains keys corresponding to the names of linting rules (see https://luau-lang.org/lint) or `"*"` that means "all rules"; the values must be one of "disabled", "enabled", "fatal".
- `"globals"`: extra global values; points to an array of strings where each string names a global that the type checker and linter must assume is valid and of type `any`

Example of a valid .luaurc file:

```json5
{
	"mode": "nonstrict",
	"lint": {
		"*": "fatal",
		"LocalUnused": "disabled",
	},
	"globals": ["expect"] // TestEZ
}
```

## Drawbacks

The introduction of configuration files means that it's now impossible to type check or lint sources in isolation, which complicates the code setup.

File-based JSON configuration may or may not map cleanly to environments that don't support files, such as Roblox Studio.

Support for two ways to specify the mode, "mode" and "language"/"mode", complicates specification a bit.

Using JSON5 instead of vanilla JSON limits the interoperability.

Making the default set of linting passes fatal requires enumerating them one by one, as "*": "fatal" means "enable all linting passes and make them fatal" (unlike -Werror).

Making the type checking issues visible but non-fatal is impossible.

## Alternatives

It's possible to consider forcing users to specify the source settings via `--!` comments exclusively. This is problematic as it may require excessive amounts of annotation though, which this proposal aims to simplify.

The format of the configuration file does not have to be JSON; for example, it can be a valid Luau source file which is the approach luacheck takes. This makes it more difficult to repurpose the .luaurc file to use third-party processing tools though, e.g. a package manager would need to learn how to parse Luau syntax to store configuration in .luaurc.
