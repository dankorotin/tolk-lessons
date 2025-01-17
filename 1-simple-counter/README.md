# Your first smart contract

A **smart contract** is a program running on the TON Blockchain via its [TVM](https://docs.ton.org/v3/documentation/tvm/tvm-overview) (TON Virtual Machine). It consists of **code** (TVM instructions) and **data** (persistent state) stored at a specific **address**.

> This address is derived from the initial code and data, so it can be calculated prior to deployment. Moreover, if your code produces the exact same bytecode and data on deployment as another deployed contract, their addresses will be the same. You will most likely encounter this behavior if you don't change any significant parts of this tutorial's code: after deployment, you'll see that your "new" smart contract already has transactions and a non-zero total value.

In the following tutorial, we'll create a simple counter contract, deploy it to the testnet, and query for the current total value. The logic will be very basic: it increments a counter by a specified number from an **internal message** and saves the total in its persistent data.

> Internal messages are the ones sent between blockchain entities and cannot be sent from off-chain. **These methods consume gas** (blockchain currency that's paid for transactions, code execution, and storage).

A dedicated `get` method (implemented and explained later in the tutorial) allows querying the current total.

> **`Get` methods are free** (meaning no gas is paid), as they don't mutate the blockchain state and are handled off-chain.

> For those with some experience in TON development: we will not be checking for the existence of opcodes and flags in the incoming message to keep this tutorial simple, but we will add an `assert` to ensure there is enough data to convert to a number.

## Creating the project

We will use [Blueprint](https://github.com/ton-community/blueprint) to simplify and streamline the setup, so open a terminal window (a separate one or in your IDE), navigate to the directory with your TON projects, and run the following command:

```bash
npm create ton@latest
```

This command creates a project from template. Make the following choices: name the project whatever you like (e.g., `my-project`), first contract: `Counter`, and choose the template: `An empty contract (Tolk)`. Your console should look like this:

```bash
? Project name my-project
? First created contract name (PascalCase) Counter
? Choose the project template An empty contract (Tolk)
```

Open the project directory (named `my-project` or whatever name you chose) and take a look at the contents. Depending on the IDE you're using, there may be more or fewer directories, but for now, we'll focus on just one: `contracts`. Navigate to it, and inside you'll see the only file: `counter.tolk`. Open it in the IDE and take a look at the code.

## Reading the data

The only **function** generated for you by Blueprint will look like this:
```tolk
fun onInternalMessage(myBalance: int, msgValue: int, msgFull: cell, msgBody: slice) {

}
```

> If you're coming from the FunC world, you may be surprised by the absence of the `impure` specifier. In Tolk, all functions are [impure by default](https://docs.ton.org/v3/documentation/smart-contracts/tolk/tolk-vs-func/in-detail). You can explicitly annotate a function as `pure`, and then impure operations are forbidden in its body (exceptions, global modifications, calling non-pure functions, etc.).

As mentioned above, your smart contract consists of **code** (functions like this) and **data** (stored in so-called cells). Let's deal with the former first. This particular function (part of your contract's **code**) is the one called when a smart contract is accessed **on-chain** (for example, by another smart contract) by sending a **message**. This type of message is called **internal**, hence the function name. Let's take a closer look at its parameters, especially the last two: `msgFull` and `msgBody`.

- `msgFull` has the type `cell`. **Cells are data structures that can hold up to 1023 bits of information and have links to up to 4 other cells.** This allows for creating "trees" of cells, potentially storing as much data as you need. To read data from cells, you need to begin parsing them.
- `msgBody` is a `slice`. **A slice is a representation of a cell that you can read data from.** It has a cursor that moves forward as you read data from the slice. Here, it's already set to the beginning of the message body—just what we will need very soon.

Your smart contract also has its own storage (**data**): a root cell stored in the so-called `c4` register (potentially having links to more cells if you need to store more than 1023 bits). Let's start by reading the data it consists of.

> **If you ever get confused, you can always open the `counter.tolk` file in this tutorial's repository and take a look at the final implementation. The code there is commented in detail, helping you understand what's going on and why.**

Add the following line inside the function:
```tolk
var dataSlice = getContractData().beginParse();
```

**`var` means that this variable (`dataSlice`) is *mutable*, i.e., it can (and will) be changed.** `getContractData()` reads the root cell in `c4` (it's still a `cell` at this point), and `beginParse()` makes it a `slice` we can read from. **This is exactly the reason we declared it as mutable: the cursor will move as we read data, mutating the variable.**

Now, add the following line of code below the previous one:

```tolk
var total = dataSlice.loadUint(64);
```

**This variable is also mutable, but for a different reason: we will increase it by the value we received in the message body.** Calling `loadUint(64)` on the slice we read in the first line of code makes the cursor in it start moving bit by bit, reading up to 64 bits (if available) and converting them to an *unsigned integer* (one that cannot be negative).

> 64 bits means it can have a maximum value of 2 to the power of 64 minus one, which is a *very, very* large number (18 446 744 073 709 551 615). We need to subtract one since the first possible value is 0, not 1.

Finally, we get to reading the value passed in the message body. Add the following line:

```tolk
val toAdd = msgBody.loadUint(16);
```

**`val` here means that the `toAdd` variable is *immutable*, i.e., it will be initialized once and will not change.** `msgBody` is one of the function parameters we discussed above. It has the `slice` type, so we can read from it right away, unlike the contract data (a `cell`). **We expect the body to contain 16 bits of information that we treat as a 16-bit unsigned integer**, so we read it with `loadUint(16)` and assign it to `toAdd`.

> A 16-bit unsigned integer can have a maximum value of 2 to the power of 16 minus one (65 535). You can choose it to be larger, of course. The motivation behind using 16 bits is that it will take a very long time to reach the maximum value of the counter (the 64-bit unsigned integer stored in the contract's cell that we read from above).

## Increasing the counter

Your code should look like this at the moment:

```tolk
fun onInternalMessage(myBalance: int, msgValue: int, msgFull: cell, msgBody: slice) {
    var dataSlice = getContractData().beginParse();
    var total = dataSlice.loadUint(64);
    val toAdd = msgBody.loadUint(16);
}
```

**We've read all the data we need (the current counter value, stored in a mutable variable `total`, and the value to add, stored in the immutable one—`toAdd`).** Let's increase the `total` and save it to the contract's persistent storage. Add the following lines to your existing code:

```tolk
total += toAdd;
var cellBuilder = beginCell();
cellBuilder.storeUint(total, 64);
val finalizedCell = cellBuilder.endCell();
setContractData(finalizedCell);
```

Here's what happens here, step by step:
1. **`total` gets increased** by the value of `toAdd` (you could alternatively write it as `total = total + toAdd`).
2. **A cell builder** is created by calling `var cellBuilder = beginCell()`. `builder` is the third "flavor" of data representation, in addition to `cell` and `slice`. A `cell` **stores data**, a `slice` lets you **read** from it, and a `builder` lets you **create and modify data** that you will later "pack" as a `cell`. We declare it as a `var` because it will be mutated (i.e., some data will be stored in it, as shown in the next step).
3. Calling `cellBuilder.storeUint(total, 64)` **stores the `total` value as 64 bits** in the `builder` (the future `cell`).
4. `val finalizedCell = cellBuilder.endCell()` declares an immutable variable of type `cell` (this is what you get by calling `endCell` on a `builder`).
5. Finally, `setContractData(finalizedCell)` **saves the data (a `cell`) from `finalizedCell` into the contract's storage (replacing the `cell` from the `c4` register that we read from at the very beginning of the function)**.

## Checking the input data

By now, we have assumed that the internal messages the contract receives will contain **valid data in the body (i.e., at least 16 bits of data we will treat as a 16-bit integer)**. We can formalize this assumption as an **assertion** and **throw an exception** (i.e., halt the execution and return an error code) if the amount of data is not sufficient to proceed.

> Remember: any code execution on TVM costs gas (essentially, money). Thus, the earlier a faulty computation is interrupted, the less money you (or the function's caller) spend.

Add this line of code at the very beginning of the function, above the line where we begin parsing the contract's data:

```tolk
assert(msgBody.getRemainingBitsCount() >= 16, 9);
```

Here are the details:
- **Declare an `assert` with two arguments**: the condition and the exception code.
- The **condition** is: “count the bits left in the message body and make sure the amount is 16 or more” (it doesn't move the slice's cursor).
- The **exception code** is 9. You can choose any code you like, but it’s better to stick to conventions. This one is found in the [TVM Whitepaper](https://ton-blockchain.github.io/docs/tvm.pdf) and means “Cell underflow: Deserialization error”.

[//]: TODO: (As you only have one script &#40;`deployCounter.ts`&#41; there will be no prompts regarding what to run, but that is the script being executed. In particular, this command...)

# 🚧 Work in progress 🚧 #