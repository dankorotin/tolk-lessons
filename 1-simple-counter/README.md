# Your first smart contract

A smart contract is a program running on the TON Blockchain via its [TVM](https://docs.ton.org/v3/documentation/tvm/tvm-overview) (TON Virtual Machine). It consists of code (TVM instructions) and data (persistent state) stored at a specific address.

> This address is derived from the initial code and data, so it can be calculated prior to deployment. Moreover, if your code produces the exact same bytecode and data on deployment as another deployed contract, their addresses will be the same. You will most likely encounter this behavior if you don't change any significant parts of this tutorial's code: after deployment, you'll see that your "new" smart contract already has transactions and a non-zero total value.

In the following tutorial, we'll create a simple counter contract, deploy it to the testnet, and query for the current total value. The logic will be very basic: it increments a counter by a specified number from an internal message and saves the total in its persistent data.

> Internal messages are the ones sent between blockchain entities and cannot be sent from off-chain. These methods consume gas (blockchain currency that's paid for transactions, code execution, and storage).

A dedicated `get` method (implemented and explained later in the tutorial) allows querying the current total.

> `Get` methods are free (meaning no gas is paid), as they don't mutate the blockchain state and are handled off-chain.

For those with some experience in TON development: we will not be checking for the existence of opcodes and flags in the incoming message to keep this tutorial simple, but we will add an `assert` to ensure there is enough data to convert to a number.

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

## Let's dive in!

The only function generated for you by Blueprint will look like this:
```tolk
fun onInternalMessage(myBalance: int, msgValue: int, msgFull: cell, msgBody: slice) {

}
```

> If you're coming from the FunC world, you may be surprised by the absence of the `impure` specifier. In Tolk, all functions are [impure by default](https://docs.ton.org/v3/documentation/smart-contracts/tolk/tolk-vs-func/in-detail). You can explicitly annotate a function as `pure`, and then impure operations are forbidden in its body (exceptions, global modifications, calling non-pure functions, etc.).

As mentioned above, your smart contract consists of **code** (functions like this) and **data** (stored in so-called cells). Let's deal with the former first. This particular function (part of your contract's **code**) is the one called when a smart contract is accessed on-chain (for example, by another smart contract). This type of message is called "internal," hence the function name. Let's take a closer look at its parameters, especially the last two: `msgFull` and `msgBody`.

- `msgFull` has the type `cell`. Cells are data structures that can hold up to 1023 bits of information and have links to up to 4 other cells. This allows for creating "trees" of cells, potentially storing as much data as you need. To read data from cells, you need to begin parsing them.
- `msgBody` is a `slice`. A slice is a representation of a cell that you can read data from. It has a cursor that moves forward as you read data from the slice. Here, it's already set to the beginning of the message body—just what we will need very soon.

Your smart contract also has its own storage (**data**): a root cell stored in the so-called `c4` register (potentially having links to more cells if you need to store more than 1023 bits). Let's start by reading the data it consists of.

Add the following line inside the function:
```tolk
var dataSlice = getContractData().beginParse();
```

> If you ever get confused, you can always open the `counter.tolk` file in this tutorial's repository and take a look at the final implementation. The code there is commented in detail, helping you understand what's going on and why.

`var` here means that this variable (`dataSlice`) is *mutable*, i.e., it can (and will) be changed. `getContractData()` reads the root cell in `c4` (it's still a `cell` at this step), and `beginParse()` makes it a `slice` we can read from. This is exactly the reason we declared it as mutable: the cursor will move as we read data, mutating it.


[//]: TODO: (As you only have one script &#40;`deployCounter.ts`&#41; there will be no prompts regarding what to run, but that is the script being executed. In particular, this command...)

# 🚧 Work in progress 🚧 #