# IL Editing
IL Editing is the process of editing a method's CIL with MonoMod using `ILHook`s, this allows for finer more granular control than standard `Hook`s (commonly referred to as detours) provide.

tModLoader provides generated events for hooking vanilla methods (usually `On`/`IL_TypeName.MethodName`.)
If a method is not provided by these events you should use `MonoModHooks.Add`/`Modify`to apply hooks, (`Add` being for detours whilst `Modify` being for IL edits.)\
Hooks applied in the manner listed above do NOT have to be unloaded manually.

Although this guide is targeted at tModLoader mod developers, most information is applicable to all projects that use MonoMod.

# Prerequisites
- [ILSpy](https://github.com/icsharpcode/ilspy) - Allows for browsing of disassembly, with options to show C# lines above their respective instructions.
- Familiarity with tModLoader and Terraria's codebase.

# Concepts
## The Stack
The stack is a collection of values that can be added to (pushed,) or removed from (popped;) instructions may push values to and from the stack.\
The stack is a first-in, last-out list where the most recently pushed value is the first to be popped.

**TODO:** Further reading

## Instructions/Opcodes
An Operation Code (Opcode) also called an instruction, specifies what operation to preform.

> [!IMPORTANT]
> It is vital that you understand what each Opcode does when writing edits, ILSpy gives a tooltip when hovering an instruction and will open documentation when clicked.\
> <img src="ILSpyTooltip.png">

### Examples:
<table>
<tr><th>C#</th><th>IL</th><th>Stack</th></tr>
<tr>
<td valign="top">  

```cs
Whatever.SomeMethod(
    0,
    "Hello",
    new SomeType()
);
```

</td>
<td valign="top">

```cs
// Load 0 onto the stack
IL_0000 ldc.i4.0
// Load the string "Hello" onto the stack
IL_0001 ldstr "Hello"
// Creates a new object of SomeType, and pushes it to the stack
IL_0002 newobj instance void Whatever.SomeType::.ctor()
// Calls the static method SomeMethod and pops
// the arguments off of the stack
IL_0003 call void Whatever::SomeMethod(int32, string, class Whatever.SomeType)
```

</td>
<td valign="top">


```cs
.
(0)

("Hello", 0)

(SomeType, "Hello", 0)


()
```

</td>
</tr>
</table>

<table>
<tr><th>C#</th><th>IL</th><th>Stack</th></tr>
<tr>
<td valign="top">  

```cs
var x = MathF.Sin(
    Main.GlobalTimeWrappedHourly
);

x /= 2f;

x += 0.5;
```

</td>
<td valign="top">

```cs
// Loads the static field Main.GlobalTimeWrappedHourly
IL_0000 ldsfld float32 Main::GlobalTimeWrappedHourly
// Pops the top value off of the stack to MathF.Sin
// and pushes the output to the stack
IL_0001 call float32 MathF::Sin(float32)
// Pushes the current value on the stack to the
// local 'x' represented by the index 0
IL_0002 stloc 0
// Loads the local at index 0
IL_0003 ldloc 0
// Load a float32 with the value 2 onto the stack
IL_0004 ldc.r4 2
// Divides the 2nd value on the stack by the top value
// popping them and pushing the result
IL_0005 div
IL_0006 stloc 0
IL_0003 ldloc 0
IL_0004 ldc.r4 0.5
// Adds the top value on the stack to the 2nd value
// popping them and pushing the result
IL_0005 add
IL_0006 stloc 0
```

</td>
<td valign="top">

```cs
.
(3911.4)


(-0.75011...)


()

(-0.75011...)

(2, -0.75011...)


(-0.375055...)
()
(-0.375055...)
(0.5, -0.375055...)


(0.124945...)
()
```

</td>
</tr>
</table>

## ILCursor
The `ILCursor` type is the standard way of manipulating `ILContext`, can be initialized directly from the `ILContext` instance provided in edits.
`ILCursor` provides various methods for navigating the context, namely (`Try`)`GotoNext`/`Prev`and `FindNext`/`Prev`, these methods should always be used instead of manually setting `ILCursor.Index`.

Use `ILCursor.Emit` or `ILCursor.Emit[X]` (With `[X]` as the name of the Opcode) to insert an instruction at the index of the cursor, this will move the cursor after the emitted instruction as well.

`ILCursor.Remove`(`Range`) removes the next instruction(s), should NOT be used for compatibility with other mods, (although context dependant.)

## Targets/Labels
A target/label points to a specific instruction in the context, used by Branching opcodes to move to the target.\
`ILLabel`s can be obtained from the context by outing them from match predicates in navigation methods, or created with `ILCursor.DefineLabel`.
```cs
ILLabel? jumpCheckTarget = null;

c.GotoNext(
    i => i.MatchBneUn(out jumpCheckTarget)
);

if (jumpCheckTarget is null)
{
    return;
}
```
When matching instructions be aware of incoming labels that point to the next instruction, use `ILCursor.MoveAfterLabels` to have the cursor redirect incoming labels to the newly emitted instruction.\
### Example:
Say we have the following context:

<table>
<tr><th>C#</th><th>IL</th></tr>
<tr>
<td valign="top">  

```cs
if (Whatever.SomeStaticBool)
{
    Whatever.Cool0ArgMethod();
}
// - Place we want to insert into.
Whatever.CoolMultipleArgMethod(0, 1, 2);
```

</td>
<td valign="top">

```cs
// if (Whatever.SomeStaticBool)
IL_0000 ldsfld bool Whatever::SomeStaticBool
IL_0001 brfalse IL_0003

// Whatever.Cool0ArgMethod();
IL_0002 call void Whatever::Cool0ArgMethod()

// Whatever.CoolMultipleArgMethod(0, 1, 2);
IL_0003 ldc.i4.0 
IL_0004 ldc.i4.1
IL_0005 ldc.i4 2
IL_0006 call void Whatever::CoolMultipleArgMethod(int32, int32, int32)
```

</td>
</tr>
</table>

Note the incoming label on `IL_0003`.
If we were to match like so:
```cs
c.GotoNext(
    MoveType.Before,
    i => i.MatchLdcI4(out _)
);
```
we would be placed before `IL_0003`, but any newly emitted instructions would be placed before the label, you can think of this as us emitting inside the `if` statement.
You can use `ILCursor.MoveAfterLabels` to emit instructions correctly.

> [!NOTE]
> `MoveType.AfterLabels` is also applicable here as it acts the same as matching before the instructions and moving after labels.

# Limitations
## In-lining
Certain methods, particularly smaller methods that are not decorated with `[MethodImpl(MethodImplOptions.NoInlining)]` may be inlined,
in which methods invoking the target method will replace the target method with its body directly, unapplying any hooks.
This manifests seemingly randomly and can differ from session to session.

This however can be rectified by applying a blank edit to all methods that use the target method (found easily by using ILSpy's analyze feature) after applying hooks to the target method.
```cs
On_FilterManager.CanCapture += (_, _) => true;

// "Re-JITs" Main.DoDraw, causing the method to consider FilterManager.CanCapture with our hooks applied.
IL_Main.DoDraw += _ => { };
```
## Garbage Collection
TODO

# Best Practices
- It is preferred that edits are done in static methods.
- Edits should NOT be segregated to seperate types/files that handle specifically monomod behaviour if it can be helped.
- Edits should be written with other mods in mind, don't be entirely reliant on long sequences of predicates matching,
  and instructions should not be removed as other mods may rely on them for their own matching.
- Opt for handling large logic inside calls or delegates instead of emitting the entire method body manually.
- When emitting instructions pertaining to locals/arguments, you should prefer grabbing the index of the local/argument from the context similarly to labels.
  ```cs
  var elementIndex = -1;

  c.GotoNext(
      MoveType.After,
      i => i.MatchLdarg(out elementIndex),
      i => i.MatchLdarg(out _),
      i => i.MatchCall<UIElement>("DrawSelf")
  );
  ```

# Simple Example
- Outline a simple injection of custom logic into some method.

# Complex Example
- Outline injection of custom logic into some method that makes use of branching.

# Debugging
- Outline common errors and what causes them.
