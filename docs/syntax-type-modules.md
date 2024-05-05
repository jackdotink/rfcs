# Type Modules

Type modules are collections of types grouped under a name. They can be used to better organize types and avoid name conflicts.

## Motivation

Currently types are a flat dictionary of names to types in each file. This leads to the pattern of prefixing an organizational label to each type to group related types together when exporting a large number of types through a single file (as is commonly done in libraries). This is cumbersome, error-prone, and generally not a good solution (see C).

This RFC adds a new syntax for grouping types together within a module. This allows for better organization of types and avoids name conflicts. Additionally, type modules already exist in the language, but are not user-definable and only created from `require` calls. This RFC makes type modules user-definable.

## Design

### Creating Type Modules

Type module are creating using `type module` followed by the module's name and a block of types. For example:

```lua
type module M
	type T = number
end
```

This creates a type module named `M` with a single type `T`. The type module can then be used like so:

```lua
local v: M.T = 5
```

### Nested Type Modules

Type modules can be nested within each other. For example:

```lua
type module OutsideModule
	type module InsideModule
		type T = number
	end
end

local v: OutsideModule.InsideModule.T = 5
```

### Scoping Rules

The module path of any type is always scoped to the current file.

```lua
type module M
	type T = number
	type U = M.T
end
```

### Type Modules Across Files

As it stands currently, `require` already creates a type module from the exported types within the required file, and names it to the variable that the file is required into. This proposal does not change that behavior.

```lua
-- A.luau
export type T = number

-- B.luau
local A = require("A")

local v: A.T = 5
```

However if a type module is created in a file, it is exported if the type module contains any exported types.

```lua
-- A.luau
type module M
	export type T = number
	type U = string -- this type cannot be used outside of this file
end

type module N
	-- this module cannot be accessed outside of this file because it contains no exported types
	type T = number
end

-- B.luau
local A = require("A")

local v: A.M.T = 5
local w: A.N.T -- type error: A.N.T does not exist in this scope
```

These same exporting rules apply to nested type modules.

There are some cases where it is desired that a type module is re-exported from another file, which can be done by using `export type module`:

```lua
local A = require("A")

-- you could re-export an entire file's module like so
export type module A = A

-- or you could re-export a specific type module like so
export type module A_M = A.M

-- this has the side effect as being able to export nested type modules
type module M
	type module N
		export type T = number
	end
end

export type module M_N = M.N
```

This will only export types within the module that are exported types.

## Drawbacks

This proposal adds new syntax to the language, which always comes with extra parsing complexity and cognitive overhead. Additionally, this proposal does not add any new functionality to the language, as users can continue prefixing type names to organize them.

## Alternatives

The main alternative is to not add this feature to the language. This would mean that users would have to continue prefixing type names to organize them.

For some smaller parts of this proposal, there are some alternatives:

- Instead of scoping from the file, scoping could instead be done from the current module. This would require additional syntax to be added for bringing types from other type modules into scope.
- Instead of exporting type modules that contain exported types, the type module could be exported as a whole. This would require either all the types within a type module to adopt the exported status of the module, or for exported types in a non-exported module to be an error. As modules exist solely for organization, giving them an export status would be misleading.
