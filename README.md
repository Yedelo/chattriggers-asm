# ChatTriggers (2.2) ASM injection
[Original ASM injection guide](https://github.com/ChatTriggers/ChatTriggers/wiki/ASM-Injection)

# Introduction

ChatTriggers provides many useful features such as triggers and libraries like ChatLib. It also provides access to most of Forge's features by accessing their classes and subscribing to normal Forge events.
While you can do a lot with these features, you might want to do something that was not intended or designed before. You can do some of these, like making custom events for chat messages, but eventually you might have to modify game code to achieve your goals.

Although ASM is powerful, try not to use it at all. It can be both risky (and produce weird code) and needs a game restart when other people want to install your module. It can also be frustrating and risky for your sanity.

## How it goes

LaunchWrapper is a system that Mojang made to fix code in older versions of Minecraft without changing the game itself. It provides a tweaker system, which can do numerous things. Most importantly it gives tweak authors the ability to transform raw classes (byte arrays).
```java
public byte[] transform(String name, String transformedName, byte[] basicClass);
```
Forge uses a Tweaker to load and also gives this transformation ability to mods with loading plugins. ChatTriggers uses this to add some custom events (such as the `messageSent` event) using the [ASMHelper library](https://github.com/ChatTriggers/ChatTriggers/wiki/ASM-Injection). It also gives us the ability to transform classes (although more limited) with the context of this library.

# Now we go

Normal module code is ran somewhat early when Minecraft starts. However, since we need to transform classes, ASM entry code needs to run earlier. Since we run before any of Minecraft starts we cannot access any of it's code or our module's code directly.

In your module, add an asmEntry as such (replace `asm.js` with whatever file name you need) to your `metadata.json`:
```json
    "asmEntry": "asm.js" 
```
Then, make the new `asm.js` file:
```js
export default ASM => {

}
```
The `ASM` object is the ASM lib we use for adding injections and also contains references to some useful class names we might need.
Yyou can try running some code in this ASM function.
```js
export default ASM => {
    const System = Java.type("java.lang.System");
    System.out.println("Hello ASM world!");
}
```
`
[00:37:33] [main/INFO] [STDOUT]: [sun.reflect.NativeMethodAccessorImpl:invoke0:-2]: Hello ASM world!
`
This code runs early and it can be useful in some cases (?) but we want to modify classes. The most powerful class modification is modifying method bodies with our own bytecode but for now, let's make some more simple modifications. 

## Adding new fields

```js
export default ASM => {
    const {AccessType} = ASM; // the ASM object has many useful variables and such

    const className = "net/minecraft/client/Minecraft";
    const fieldName = "attacks";
    const fieldDescriptor = "I":
    const accessType = AccessType.PUBLIC;

    const initialValue = 0;

    ASM.fieldBuilder(
        className,
        fieldName,
        fieldDescriptor,
        accessType
    ).
    initialValue(initialValue).
    execute();
}
```

The `className` is the full name of the class we add the field to, with `/`'s instead of `.`'s for package seperators.
The `fieldName` is the name of the field. 
The `fieldDescriptor` is the descriptor of the field. You can learn more about field descriptors [here](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-4.html#jvms-4.3.2) with some more information below. In this case the field is an integer, so the descriptor is a simple `I`.
The `accessType` is a custom ASMHelper enum. This should always be `PUBLIC` but it could also be `PRIVATE` or `PROTECTED` if you want (why).

The `initialValue` is what the field originally starts as.

`execute` must be called after all builders or else the code won't take effect.

Now we can use the field `attacks` as we would use any other field in the Minecraft class.
```js
register("attackEntity", () => {
    Client.getMinecraft().attacks ++;
    ChatLib.chat("Total attacks: " + Client.getMinecraft().attacks);	
})
```
![image](total attacks screenshot.png)
