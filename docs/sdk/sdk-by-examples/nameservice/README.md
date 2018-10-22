# SDK By Examples - Nameservice

In this tutorial, you will build a simplistic but functional decentralised application using the [Cosmos SDK](https://github.com/cosmos/cosmos-sdk/) and learn the basics so that you can get started building your own modules and decentralized applications.  

The application we will build is a "nameservice", a mapping of strings to other strings (similar to [Namecoin](https://namecoin.org/), [ENS](https://ens.domains/), or [Handshake](https://handshake.org/)), in which to buy the name, the buyer has to pay the current owner more than the current owner paid to buy it!

All of the final source code for this tutorial project is in this directory, however, it is highly recommended that you follow along manually and try building the project yourself!

## Tutorial parts

1. [Design](#the-application) your application
    * [State](#state)
    * [Messages](#messages)
2. Start by building your [`Keeper`](#the-keeper.md)
3. Define interactions with your chain through [`Msgs` and `Handlers`](#messages-and-handlers)
    * [`SetName`](#set-name.md)
    * [`BuyName`](#buy-name.md)
4. Make views on your state machine with [`Queriers`](#querier.md)
5. Register your types in the encoding format using [`sdk.Codec`](#codec-file)
6. Create [CLI interactions for your module](#nameservice-module-cli)
7. Put it all together in [`./app.go`](#app.go)!
8. Create the [`nameshaked` and `nameshakecli` entry points](#entrypoints)
9. Setup [dependencies](#dependencies)
10. [Install and run](#installing-the-software) the software

We will start by describing the design of our application. If you want to jump directly to the coding section, click [here](#project-directory-structure).

## The application

The goal of the application is to let anyone buy names and set a value this name resolves to. The value is set by the owner, and the owner is the current highest bidder for a given name. 

Let us try to  understand how these simple requirements translate in terms of application design. Be sure to read the [prerequisite material](../intro#prerequisite-reading) if you haven't already.

A decentralised application is just a replicated deterministic state-machine. As a developer, you just have to define the state-machine (i.e. a state and messages that trigger state-transititons), and *Tendermint* will replicate it for you. 

The Cosmos-SDK is there to help you build this state-machine. The SDK is a modular framework, meaning applications are built by aggregating a collection of interoperable modules. Each module represents its own little message processor, and the SDK is responsible for routing each message in its respective module. 

Let us list the modules we need for our nameservice application:
- `auth`: This module is needed to handle accounts and fees.
- `bank`: This module enables us to have tokens in our application.
- `nameservice`: This module does not exist yet! It will handle the logic for our nameservice. It's the main piece of software we have to work on to build our application. 

Now, let us look at the two main parts of our application, the state and the message types. 

### State

The state represents your application at a given moment. It tells how much token each account possesses, what are the owners and price of each name, and to what value each name resolves to. 

The state of tokens and accounts is defined by the `auth` and `bank` modules, which means we don't have to concern ourselves with it for now. What you need to do is define the part of the state that relates specifically to your nameservice module.

In the SDK, everything is stored in one store called the `multistore`. Any number of KVStores can be created in this multistore. For your application, you need to store:

- A mapping of `name` to `value`. We will create a `nameStore` in the `multistore` for it.
- A mapping of `name` to `owner`. We will create a `ownerStore` in the `multistore` for it.
- A mapping of `name` to `price`. We will create a `priceStore` in the `multistore` for it.

### Messages

Messages are contained in transaction, and trigger state-transitions. Each module defines a list of messages and how to handle them. Here are the messages you need for your nameservice application:

- `MsgSetName`: This message allows name owners to set a value for a given name in the `nameStore`.
- `MsgBuyName`: This message allows accounts to buy a name and become their owner in the `ownerStore`. 

When a transaction (included in a block) reaches a Tendermint node, it is passed to the application via the ABCI and decoded to get the message. The message is then routed to the appropriate module and handled there according to the logic defined in the `handler`. If the state needs to be updated, the `handler` calls the `keeper` to perform the update.

Now that you have defined how your application works from a high-level perspective, you can start implementing it! 

## Project directory structure

Through the course of this tutorial you will create the following files that make up your application:

```bash
./nameservice
├── Gopkg.toml
├── app.go
├── cmd
│   ├── nameshakecli
│   │   └── main.go
│   └── nameshaked
│       └── main.go
└── x
    └── nameservice
        ├── client
        │   └── cli
        │       ├── query.go
        │       └── tx.go
        ├── codec.go
        ├── handler.go
        ├── keeper.go
        ├── msgs.go
        └── querier.go
```

## The Keeper

**[Reference implementation](./x/nameservice/keeper.go)**

The main core of a Cosmos SDK module is a piece called the Keeper. It is what handles interaction with your module's stores, has references to other module's keepers, and often contains most of the core functionality of a module.  

Begin by creating the file ./x/nameservice/keeper.go to hold the keeper for your module. In Cosmos SDK applications the convention is that modules live in the ./x/ folder.

### Keeper Struct

In the `keeper.go` file, start by placing the following code.

```go
package nameservice

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/cosmos/cosmos-sdk/x/bank"
)

// Keeper - handlers sets/gets of custom variables for your module
type Keeper struct {
	coinKeeper bank.Keeper

	namesStoreKey sdk.StoreKey // The (unexposed) key used to access the store from the Context.
    ownersStoreKey sdk.StoreKey // The (unexposed) key used to access the store from the Context.
    priceStoreKey sdk.StoreKey // The (unexposed) key used to access the store from the Context.

    cdc *codec.Codec // The wire codec for binary encoding/decoding.
}
```

A couple of notes about the above code:

- The package name `nameservice` is the name of the package that this file is part of.  In Go, all code has to be part of a package.
- Note the `import` of 3 different `cosmos-sdk` packages:
    * [`codec`](https://godoc.org/github.com/cosmos/cosmos-sdk/codec) - provides tools to work with the Cosmos encoding format, [Amino](https://github.com/tendermint/go-amino)
    * [`bank`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank) - the bank module controls accounts and coin transfers
    * [`types`](https://godoc.org/github.com/cosmos/cosmos-sdk/types) - contains commonly used types throughout the SDK
- The `Keeper` struct. In this keeper there are a couple of key pieces:
    * `bank.Keeper` - This is a reference to the keeper from the bank module.  It allows code in this module to be able to call functions from the bank's keeper. The SDK uses an object capabilities approach to accessing parts of the sections of the application state. This is to allow developers to employ a least authority approach limiting the capabilities of a faulty or malicious module from affecting parts of state it doesn't need access to.
    * `*codec.Codec` - This is a pointer to the codec that is used by Amino to encode and decode binary structs.
    * `sdk.StoreKey` -  This gates access to a `sdk.KVStore` that persists state from transactions into the underlying network.
- This module has 3 store keys:
    * `namesStoreKey` - This is the main store that stores the value string that the name points to (i.e. `map[name]value`)
    * `ownersStoreKey` - This store contains the current owner of this name (i.e. `map[sdk_address]name`)
    * `priceStoreKey` - This store contains the price that the current owner paid. Anyone buying this name must spend more than the current owner. (i.e. `map[name]price`)

### Getters and Setters

Now it is time to add methods to interract with the store via the keeper. First, add a function to set the string a given name resolves to.

```go
// SetName - sets the value string that a name resolves to
func (k Keeper) SetName(ctx sdk.Context, name string, value string) {
	store := ctx.KVStore(k.namesStoreKey)
	store.Set([]byte(name), []byte(value))
}
```

This method first gets the store object for the name resolutions using the `namesStoreKey` from the Keeper.

>NOTE: This function uses the [`sdk.Context`](https://godoc.org/github.com/cosmos/cosmos-sdk/types#Context) object. This object holds functions to access a number of import pieces of state like blockHeight and chainID. See the godoc for more information on the methods it exposes.

Next, it inserts the `<name, value>` pair into the store using the store's `.Set([]byte, []byte)` method. As the store only takes `[]byte` type and we want to store a `string`, we first need to cast the `string` to `[]byte` in order to use them as parameters for the `Set` method.

Now, add a method to actually resolve the names.

```go
// ResolveName - returns the string that the name resolves to
func (k Keeper) ResolveName(ctx sdk.Context, name string) string {
	store := ctx.KVStore(k.namesStoreKey)
	bz := store.Get([]byte(name))
	return string(bz)
}
```

Here, like `SetName` method, the `ResolveName` method first retrieves the store object using the `StoreKey`.  Next, it calls the `.Get([]byte) []byte` method. The key, which is the `name` string casted to `[]byte`, is passed as the parameter into the function.  The result is then casted to a `string` before being returned. .

Now,add similar functions for getting and setting `Owners`.

```go
    // GetOwner - get the current owner of a name
    func (k Keeper) GetOwner(ctx sdk.Context, name string) sdk.AccAddress {
        store := ctx.KVStore(k.ownersStoreKey)
        bz := store.Get([]byte(name))
        return bz
    }

    // SetOwner - sets the current owner of a name
    func (k Keeper) SetOwner(ctx sdk.Context, name string, owner sdk.AccAddress) {
        store := ctx.KVStore(k.ownersStoreKey)
        store.Set([]byte(name), owner)
    }

    // HasOwner - returns whether or not the name already has an owner
    func (k Keeper) HasOwner(ctx sdk.Context, name string) bool {
        store := ctx.KVStore(k.ownersStoreKey)
        bz := store.Get([]byte(name))
        return bz != nil
    }
```

A couple notes on the code:
- The data is retrieved from the `ownersStore` store instead of the `namesStore` store
- Because `sdk.AccAddress` is a type alias for `[]byte`, you can natively cast to it.
- The function `HasOwner`  tells whether a name already has an owner or not

Finally, add a getter and setter for the price of a name.

```go
    // GetPrice - gets the current price of a name.  If price doesn't exist yet, set to 1steak.
    func (k Keeper) GetPrice(ctx sdk.Context, name string) sdk.Coins {
        if !k.HasOwner(ctx, name) {
            return sdk.Coins{sdk.NewInt64Coin("mycoin", 1)}
        }
        store := ctx.KVStore(k.priceStoreKey)
        bz := store.Get([]byte(name))
        var price sdk.Coins
        k.cdc.MustUnmarshalBinary(bz, &price)
        return price
    }

    // SetPrice - sets the current price of a name
    func (k Keeper) SetPrice(ctx sdk.Context, name string, price sdk.Coins) {
        store := ctx.KVStore(k.priceStoreKey)
        store.Set([]byte(name), k.cdc.MustMarshalBinary(price))
    }
```

The data is stored in the `priceStore`.  Note that `sdk.Coins` does not have it's own bytes encoding, which means the price needs to be marsalled and unmarshalled using [Amino](https://github.com/tendermint/go-amino/) to be inserted or removed from the store. 

When getting the price for a name that has no owner (and thus no price), the method will return 1 "mycoin" as the price.

## Messages and Handlers

Now that you have the `Keeper` setup, it is time to build the `Msg` and `handlers` that actually allow users to buy and set names.

### Set Name

#### Message

Let us first setup the different messages that a user can use to interact with this module. The Cosmos SDK define a standard interface that all `Msgs` must satisfy:

```go
// Transactions messages must fulfill the Msg
type Msg interface {
	// Return the message type.
	// Must be alphanumeric or empty.
	Type() string

	// Returns a human-readable string for the message, intended for utilization
	// within tags
	Name() string

	// ValidateBasic does a simple validation check that
	// doesn't require access to any other information.
	ValidateBasic() Error

	// Get the canonical byte representation of the Msg.
	GetSignBytes() []byte

	// Signers returns the addrs of signers that must sign.
	// CONTRACT: All signatures must be present to be valid.
	// CONTRACT: Returns addrs in some deterministic order.
	GetSigners() []AccAddress
}
```

Start by defining `MsgSetName` in a new file `./x/nameservice/msgs.go`, a message that allows owner of an address to set the value a name resolves to.

```go
type MsgSetName struct {
	NameID string
	Value  string
	Owner  sdk.AccAddress
}

func NewMsgSetName(name string, value string, owner sdk.AccAddress) MsgSetName {
	return MsgBuyName{
		NameID: name,
		Value:  value,
		Owner:  owner,
	}
}
```
The `MsgSetName` has three attributes:
- `name` - The name trying to be set.
- `value` - What the name resolves to.
- `owner` - The owner of that name.

Note that the field name `NameID` is used rather than `Name` as `.Name()` is the name of a method on the `Msg` interface.  This will be resolved in a [future update of the SDK](https://github.com/cosmos/cosmos-sdk/issues/2456).

Next, let us define the `Type()` and `Name()` methods of the `Msg` interface:

```go
// Implements Msg.
func (msg MsgSetName) Type() string { return "nameservice" }
func (msg MsgSetName) Name() string { return "set_name"}
```

These methods are used by the SDK to route `msgs` to the proper module for handling and for adding human readable names to tags.

Now, let us define the `ValidateBasic()` method:

```go
// Implements Msg.
func (msg MsgSetName) ValidateBasic() sdk.Error {
	if msg.Owner.Empty() {
		return sdk.ErrInvalidAddress(msg.Owner.String())
	}
	if len(msg.NameID) == 0 || len(msg.Value) == 0 {
		return sdk.ErrUnknownRequest("Name and Value cannot be empty")
	}
	return nil
}
```
This method is used to provide some basic *stateless* checks on the validity of the msg.  In this case, we check that none of the attributes are empty.

Finally, let us define the `GetSignBytes()` and `GetSigners()` methods:

```go
// Implements Msg.
func (msg MsgSetName) GetSignBytes() []byte {
	b, err := json.Marshal(msg)
	if err != nil {
		panic(err)
	}
	return sdk.MustSortJSON(b)
}

// Implements Msg.
func (msg MsgSetName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Owner}
}
```

The first defines how the `Msg` gets encoded for signing. This should usually be in JSON and should not be modified in most cases.

The second allows the `Msg` to define whose signature is required on a transaction in order for it to be valid.  In this case, for example, the `MsgSetName` requires that the `Owner` sign the transaction trying to reset what the name points to.

#### Handler

Now that the `MsgSetName` is defined, you have to define the handler that actually executes the `Msg`.

In a new file called `handler.go` in the `nameservice` package, start off with:

```go
package nameservice

import (
	"fmt"
	"reflect"

	sdk "github.com/cosmos/cosmos-sdk/types"
)

// NewHandler returns a handler for "nameservice" type messages.
func NewHandler(keeper Keeper) sdk.Handler {
	return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
		switch msg := msg.(type) {
		case MsgSetName:
			return handleMsgSetName(ctx, keeper, msg)
		default:
			errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", reflect.TypeOf(msg).Name())
			return sdk.ErrUnknownRequest(errMsg).Result()
		}
	}
}
```
This is essentially a subrouter that directs messages coming into this module to the proper handler for the message.  At the moment, there is only one `Msg`/`Handler`.

Now, you need to handle the actual logic for handling message in `handleMsgSetName`.

```go
// Handle MsgSetName
func handleMsgSetName(ctx sdk.Context, keeper Keeper, msg MsgSetName) sdk.Result {
	if !msg.Owner.Equals(keeper.GetOwner(ctx, msg.NameID)) { // Checks if the the msg sender is the same as the current owner
		return sdk.ErrUnauthorized("Incorrect Owner").Result() // If not, throw an error
	}
	keeper.SetName(ctx, msg.NameID, msg.Value) // If so, set the name to the value specified in the msg.
	return sdk.Result{}                      // return
}
```
This function checks to see if the `Msg` sender is actually the owner of the name (which is retrieved using `keeper.GetOwner`). If so, they can set the name by calling the function on the keeper. If not, an error is thrown.

### Buy Name

Great, now owners can set names! But what if a name doesn't have an owner yet? There needs to be a way for people to buy names!

#### Msg

First, define the `Msg` for buying names and add it to the `msgs.go` file.

```go
type MsgBuyName struct {
	NameID  string
	Bid   sdk.Coins
	Buyer sdk.AccAddress
}

func NewMsgBuyName(name string, bid sdk.Coins, buyer sdk.AccAddress) MsgBuyName {
	return MsgBuyName{
		NameID:  name,
		Bid:   bid,
		Buyer: buyer,
	}
}

// Implements Msg.
func (msg MsgBuyName) Type() string { return "nameservice" }
func (msg MsgSetName) Name() string { return "buy_name"}

// Implements Msg.
func (msg MsgBuyName) ValidateBasic() sdk.Error {
	if msg.Buyer.Empty() {
		return sdk.ErrInvalidAddress(msg.Buyer.String())
	}
	if len(msg.NameID) == 0 {
		return sdk.ErrUnknownRequest("Name and Value cannot be empty")
	}
	if !msg.Bid.IsPositive() {
		return sdk.ErrInsufficientCoins("Bids must be positive")
	}
	return nil
}

// Implements Msg.
func (msg MsgBuyName) GetSignBytes() []byte {
	b, err := json.Marshal(msg)
	if err != nil {
		panic(err)
	}
	return sdk.MustSortJSON(b)
}

// Implements Msg.
func (msg MsgBuyName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Buyer}
}
```

In the `handler.go` file, add the `MsgBuyName` handler to the module router, so it now looks like this:

```go
// NewHandler returns a handler for "nameservice" type messages.
func NewHandler(keeper Keeper) sdk.Handler {
	return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
		switch msg := msg.(type) {
		case MsgSetName:
			return handleMsgSetName(ctx, keeper, msg)
		case MsgBuyName:
			return handleMsgBuyName(ctx, keeper, msg)
		default:
			errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", reflect.TypeOf(msg).Name())
			return sdk.ErrUnknownRequest(errMsg).Result()
		}
	}
}
```

Now, add the actual handle function to the `handler.go` file:

```go
// Handle MsgBuyName
func handleMsgBuyName(ctx sdk.Context, keeper Keeper, msg MsgBuyName) sdk.Result {
	if keeper.GetPrice(ctx, msg.NameID).IsGTE(msg.Bid) { // Checks if the the bid price is greater than the price paid by the current owner
		return sdk.ErrInsufficientCoins("Bid not high enough").Result() // If not, throw an error
	}
	if keeper.HasOwner(ctx, msg.NameID) {
		_, err := keeper.coinKeeper.SendCoins(ctx, msg.Buyer, keeper.GetOwner(ctx, msg.NameID), msg.Bid)
		if err != nil {
			return sdk.ErrInsufficientCoins("Buyer does not have enough coins").Result()
		}
	} else {
		_, _, err := keeper.coinKeeper.SubtractCoins(ctx, msg.Buyer, msg.Bid) // If so, deduct the Bid amount from the sender
		if err != nil {
			return sdk.ErrInsufficientCoins("Buyer does not have enough coins").Result()
		}
	}
	keeper.SetOwner(ctx, msg.NameID, msg.Buyer)
	keeper.SetPrice(ctx, msg.NameID, msg.Bid)
	return sdk.Result{}
}
```
This function checks to make sure the bid is higher than the current price.  If it is, it checks to see whether the name already has an owner. If it does, they get transferred the money from the Buyer. If it doesn't, the money just gets burned. If either `SubtractCoins` or `SendCoins` returns a non-nil error, the handler throws an error, reverting the transaction. Otherwise, the function sets the buyer to the new owner, the new price to the current bid, and returns.

Now that the core logic of your nameservice is finished, let us actually build an app that uses the module. 

## Querier

Start by creating the `./x/nameservice/querier.go` file. This is the place to define which queries against application state users will be able to make. In our `nameservice` we want to expose two:

- `resolve`: This takes a `name` and returns the `value` that is stored by the `nameservice`. This is similar to a DNS query.
- `whois`: This takes a `name` and returns the `price`, `value`, and `owner` of the name. Used for figuring out how much names cost when you want to buy them.

Start by defining the `NewQuerier` function which acts a sub-router for queries to this module (similar to our `NewHandler` function). Note that because we don't have an interface similar to `Msg` for queries we need to manually define our switch statement cases (they can't be pulled off of the query `.Name()` function):


```go
package nameservice

import (
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	abci "github.com/tendermint/tendermint/abci/types"
)

// query endpoints supported by the governance Querier
const (
	QueryResolve = "resolve"
	QueryWhois   = "whois"
)

func NewQuerier(keeper Keeper) sdk.Querier {
	return func(ctx sdk.Context, path []string, req abci.RequestQuery) (res []byte, err sdk.Error) {
		switch path[0] {
		case QueryResolve:
			return queryResolve(ctx, path[1:], req, keeper)
		case QueryWhois:
			return queryWhois(ctx, path[1:], req, keeper)
		default:
			return nil, sdk.ErrUnknownRequest("unknown nameservice query endpoint")
		}
	}
}
```
Now that the router is defined, define the inputs and responses for each query:

```go
// nolint: unparam
func queryResolve(ctx sdk.Context, path []string, req abci.RequestQuery, keeper Keeper) (res []byte, err sdk.Error) {
	name := path[0]

	value := keeper.ResolveName(ctx, name)

	if value == "" {
		return []byte{}, sdk.ErrUnknownRequest("could not resolve name")
	}

	return []byte(value), nil
}

type Whois struct {
	Value string         `json:"value"`
	Owner sdk.AccAddress `json:"owner"`
	Price sdk.Coins      `json:"price"`
}

// nolint: unparam
func queryWhois(ctx sdk.Context, path []string, req abci.RequestQuery, keeper Keeper) (res []byte, err sdk.Error) {
	name := path[0]

	whois := Whois{}

	whois.Value = keeper.ResolveName(ctx, name)
	whois.Owner = keeper.GetOwner(ctx, name)
	whois.Price = keeper.GetPrice(ctx, name)

	bz, err2 := codec.MarshalJSONIndent(keeper.cdc, whois)
	if err2 != nil {
		panic("could not marshal result to JSON")
	}

	return bz, nil
}
```

Notes on the above code:
- Here our `Keeper` getters/setters come into heavy use. When building any other applications you may need to go back and define more getters/setters to access the pieces of state you need at this point.
- If your application needs some custom response types (`Whois` here), define them in this file.

## Codec File

To [register your types with Amino](https://github.com/tendermint/go-amino#registering-types) so that they can be encoded/decoded by your module there is a bit of code that needs to be placed in `./x/nameservice/codec.go`. Any interface you create and any struct that implements an interface. In this app the two `Msg` interface implementations we have (`SetName` and `BuyName`) need to be registered, but our `Whois` query return type does not:

```go
package nameservice

import (
	"github.com/cosmos/cosmos-sdk/codec"
)

// Register concrete types on wire codec
func RegisterCodec(cdc *codec.Codec) {
	cdc.RegisterConcrete(MsgSetName{}, "nameservice/SetName", nil)
	cdc.RegisterConcrete(MsgBuyName{}, "nameservice/BuyName", nil)
}
```

## Nameservice Module CLI

The CosmosSDK uses the [`cobra`](https://github.com/spf13/cobra) library for CLI interactions. This library makes it easy for each module to expose it's own commands. To get started defining the user's CLI interactions with your module create the following files:

 - `./x/nameservice/client/cli/query.go`
 - `./x/nameservice/client/cli/tx.go`

Start in `query.go`. Here, define `cobra.Command`s for each of your modules `Queriers` (`resolve`, and `whois`):

```go
package cli

import (
	"fmt"

	"github.com/spf13/cobra"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
)

type QueryResult struct {
	Value string         `json:"value"`
	Owner sdk.AccAddress `json:"owner"`
	Price sdk.Coins      `json:"price"`
}

// GetCmdResolveName queries information about a name
func GetCmdResolveName(queryRoute string, cdc *codec.Codec) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "resolve [name]",
		Short: "resolve name",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			name := args[0]
			cliCtx := context.NewCLIContext().WithCodec(cdc)

			res, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/proposals/%s", queryRoute, name), nil)
			if err != nil {
				fmt.Printf("could not resolve name - %s \n", string(name))
				return nil
			}

			fmt.Println(string(res))

			return nil
		},
	}

	return cmd
}

// GetCmdWhois queries information about a domain
func GetCmdWhois(queryRoute string, cdc *codec.Codec) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "whois [name]",
		Short: "Query whois info of name",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			name := args[0]
			cliCtx := context.NewCLIContext().WithCodec(cdc)

			res, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/whois/%s", queryRoute, name), nil)
			if err != nil {
				fmt.Printf("could not resolve whois - %s \n", string(name))
				return nil
			}

			fmt.Println(string(res))

			return nil
		},
	}

	return cmd
}
```
Notes on the above code:
- The CLI introduces a new `context`: [`CLIContext`](https://godoc.org/github.com/cosmos/cosmos-sdk/client/context#CLIContext). It carries data about user input and application configuration that are needed for CLI interaction. See the godoc for more methods.
- The `path` required for the `cliCtx.QueryWithData()` function maps directly to the names in your query router.
- The first part of the path is to differentiate the types of queries possible to SDK applications: `custom` is for queriers. The second piece is for the module name. Following that is the specific querier in the module that will be called. In these examples the fourth piece is the query. This works becuase the query parameter is a simple string. To enable more complex query inputs you need to use the second arguement of the [`.QueryWithData()`](https://godoc.org/github.com/cosmos/cosmos-sdk/client/context#CLIContext.QueryWithData) function to pass in `data`.

Once the query interactions are defined, its time to move on to the transaction generation in `tx.go`:

> _*NOTE*_: Your application needs to import the code you just wrote. Here the import path is set to this repository (`github.com/sunnya97/sdk-nameservice-example/x/nameservice`). If you are following along in your own repo you will need to change the import path to reflect that (`github.com/{{ .Username }}/{{ .Project.Repo }}/x/nameservice`).


```go
package cli

import (
	"github.com/spf13/cobra"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/cosmos/cosmos-sdk/x/auth"
	authcmd "github.com/cosmos/cosmos-sdk/x/auth/client/cli"

	"github.com/sunnya97/sdk-nameservice-example/x/nameservice"
)

const (
	flagName   = "name"
	flagValue  = "value"
	flagAmount = "amount"
)

func GetCmdBuyName(cdc *codec.Codec) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "buy-name [name] [amount]",
		Short: "bid for existing name or claim new name",
		Args:  cobra.ExactArgs(2),
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().
				WithCodec(cdc).
				WithAccountDecoder(authcmd.GetAccountDecoder(cdc))

			if err := cliCtx.EnsureAccountExists(); err != nil {
				return err
			}

			name := args[0]

			amount := args[1]
			coins, err := sdk.ParseCoins(amount)
			if err != nil {
				return err
			}

			account, err := cliCtx.GetFromAddress()
			if err != nil {
				return err
			}

			msg := nameservice.MsgBuyName{
				NameID: name,
				Bid:    coins,
				Buyer:  account,
			}

			tx := auth.StdTx{
				Msgs: []sdk.Msg{msg},
			}

			bz := cdc.MustMarshalBinary(tx)

			_, err = cliCtx.BroadcastTx(bz)

			return err
		},
	}

	return cmd
}

func GetCmdSetName(cdc *codec.Codec) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "set-name [name] [value]",
		Short: "set the value associated with a name that you own",
		Args:  cobra.ExactArgs(2),
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().
				WithCodec(cdc).
				WithAccountDecoder(authcmd.GetAccountDecoder(cdc))

			if err := cliCtx.EnsureAccountExists(); err != nil {
				return err
			}

			name := args[0]
			value := args[1]

			account, err := cliCtx.GetFromAddress()
			if err != nil {
				return err
			}

			msg := nameservice.MsgSetName{
				NameID: name,
				Value:  value,
				Owner:  account,
			}

			tx := auth.StdTx{
				Msgs: []sdk.Msg{msg},
			}

			bz := cdc.MustMarshalBinary(tx)

			_, err = cliCtx.BroadcastTx(bz)

			return err
		},
	}

	return cmd
}

```

Notes on the above code:
- The `authcmd` package is used here. [The godocs have more information on usage](https://godoc.org/github.com/cosmos/cosmos-sdk/x/auth/client/cli#GetAccountDecoder). It provides access to accounts controlled by the CLI and facilitates signing.

## App.go

Next, in the root of your project directory, create a new file called `app.go`.  At the top of the file, let's declare the package and import our dependencies.

```go
package app

import (
	"os"

	abci "github.com/tendermint/tendermint/abci/types"
	cmn "github.com/tendermint/tendermint/libs/common"
	dbm "github.com/tendermint/tendermint/libs/db"
	"github.com/tendermint/tendermint/libs/log"

	bam "github.com/cosmos/cosmos-sdk/baseapp"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"

	"github.com/cosmos/cosmos-sdk/x/auth"
	"github.com/cosmos/cosmos-sdk/x/bank"
	"github.com/sunnya97/sdk-nameservice-example/x/nameservice"
)
```

> _*NOTE*_: Your application needs to import the code you just wrote. Here the import path is set to this repository (`github.com/sunnya97/sdk-nameservice-example/x/nameservice`). If you are following along in your own repo you will need to change the import path to reflect that (`github.com/{{ .Username }}/{{ .Project.Repo }}/x/nameservice`).

Here we imported some dependencies from Tendermint, from the Cosmos SDK, and then the three modules we will use in our app: `auth`, `bank`, and `nameservice`.

Links to godocs for each package:
- [`codec`](https://godoc.org/github.com/cosmos/cosmos-sdk/codec): Functions for working with amino
- [`auth`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/auth)
- [`bank`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank)
- [`baseapp`](https://godoc.org/github.com/cosmos/cosmos-sdk): This module helps developers bootstrap CosmosSDK applications
- [`types`](https://godoc.org/github.com/cosmos/cosmos-sdk): Common types for working with SDK applications
- [`abci`](https://godoc.org/github.com/tendermint/tendermint/abci/types): Similar to the `sdk/types` module, but for Tendermint
- [`cmn`](https://godoc.org/github.com/tendermint/tendermint/libs/common): Code for working with Tendermint applications
- [`dbm`](https://godoc.org/github.com/tendermint/tendermint/libs/db): Code for working with the Tendermint database


Next declare the name and struct for your app.  In this example, the app is called Nameshake, a portmanteau of Handshake and Namecoin. 

```go
const (
	appName = "Nameshake"
)

type NameshakeApp struct {
	*bam.BaseApp
	cdc *codec.Codec

	keyMain     *sdk.KVStoreKey
	keyAccount  *sdk.KVStoreKey
	keyNSnames  *sdk.KVStoreKey
	keyNSowners *sdk.KVStoreKey
	keyNSprices *sdk.KVStoreKey

	accountMapper auth.AccountMapper
	bankKeeper    bank.Keeper
	nsKeeper      nameservice.Keeper
}
```
Next, create a constructor for a new `NameshakeApp`.  In this function your application:
- Generates `storeKeys`
- Creates `Keepers`
- Registers `Handler`s
- Registers `Querier`s
- Mounts `KVStore`s
- Sets the `initChainer`

```go
func NewNameshakeApp(logger log.Logger, db dbm.DB) *NameshakeApp {
	cdc := MakeCodec()
	bApp := bam.NewBaseApp(appName, logger, db, auth.DefaultTxDecoder(cdc))

	var app = &NameshakeApp{
		BaseApp: bApp,
		cdc:     cdc,

		keyMain:     sdk.NewKVStoreKey("main"),
		keyAccount:  sdk.NewKVStoreKey("acc"),
		keyNSnames:  sdk.NewKVStoreKey("ns_names"),
		keyNSowners: sdk.NewKVStoreKey("ns_owners"),
		keyNSprices: sdk.NewKVStoreKey("ns_prices"),
	}

	app.accountMapper = auth.NewAccountMapper(
		app.cdc,
		app.keyAccount,
		auth.ProtoBaseAccount,
	)

	app.bankKeeper = bank.NewBaseKeeper(app.accountMapper)

	app.nsKeeper = nameservice.NewKeeper(
		app.bankKeeper,
		app.keyNSnames,
		app.keyNSowners,
		app.keyNSprices,
		app.cdc,
	)

	app.Router().
		AddRoute("nameservice", nameservice.NewHandler(app.nsKeeper)).

	app.QueryRouter().
		AddRoute("nameservice", nameservice.NewQuerier(app.nsKeeper))

	app.SetInitChainer(app.initChainer)

	app.MountStoresIAVL(
		app.keyMain,
		app.keyAccount,
		app.keyNSnames,
		app.keyNSowners,
		app.keyNSprices,
	)

	err := app.LoadLatestVersion(app.keyMain)
	if err != nil {
		cmn.Exit(err.Error())
	}

	return app
}
```

The `initChainer` defines how accounts in `genesis.json` are mapped into the application state on initial chain start. The constructor registers the `initChainer` function, but it isn't defined yet. Go ahead and create it:

```go
type GenesisState struct {
	Accounts []auth.BaseAccount `json:"accounts"`
}

func (app *NameshakeApp) initChainer(ctx sdk.Context, req abci.RequestInitChain) abci.ResponseInitChain {
	stateJSON := req.AppStateBytes

	genesisState := new(GenesisState)
	err := app.cdc.UnmarshalJSON(stateJSON, genesisState)
	if err != nil {
		panic(err)
	}

	for _, acc := range genesisState.Accounts {
		acc.AccountNumber = app.accountMapper.GetNextAccountNumber(ctx)
		app.accountMapper.SetAccount(ctx, &acc)
	}

	return abci.ResponseInitChain{}
}
```

Finally add a helper function to generate an amino [`*codec.Codec`](https://godoc.org/github.com/cosmos/cosmos-sdk/codec#Codec) that properly registers all of the modules used in your application:

```go
func MakeCodec() *codec.Codec {
	var cdc = codec.New()
	auth.RegisterCodec(cdc)
	bank.RegisterCodec(cdc)
	nameservice.RegisterCodec(cdc)
	sdk.RegisterCodec(cdc)
	codec.RegisterCrypto(cdc)
	return cdc
}
```

## Entrypoints

In golang the convention is to place files that compile to a binary in the `./cmd` folder of a project. For our application we have 2 binaries that we want to create:

- `nameshaked`: This binary is similar to `bitcoind` or other cryptocurrency daemons in that it maintains peer connections and propagates transactions through the network.
- `nameshakecli`: This binary provides commands that allow users to interact with your application.


To get started create two files in the root of the project directory that will instantiate these binaries:
- `./cmd/nameserviced/main.go`
- `./cmd/nameservicecli/main.go`

## Nameshaked

Start by adding the following code to `nameserviced/main.go`:

> _*NOTE*_: Your application needs to import the code you just wrote. Here the import path is set to this repository (`github.com/sunnya97/sdk-nameservice-example`). If you are following along in your own repo you will need to change the import path to reflect that (`github.com/{{ .Username }}/{{ .Project.Repo }}`).


```go
package main

import (
    "encoding/json"
    "io"
    "os"

    "github.com/spf13/cobra"

    abci "github.com/tendermint/tendermint/abci/types"
    "github.com/tendermint/tendermint/libs/cli"
    dbm "github.com/tendermint/tendermint/libs/db"
    "github.com/tendermint/tendermint/libs/log"
    tmtypes "github.com/tendermint/tendermint/types"

    "github.com/cosmos/cosmos-sdk/server"

    app "github.com/sunnya97/sdk-nameservice-example"
)

var DefaultNodeHome = os.ExpandEnv("$HOME/.nameshaked")

var appInit = server.AppInit{
    AppGenState: server.SimpleAppGenState,
    AppGenTx:    server.SimpleAppGenTx,
}

func main() {
    cdc := app.MakeCodec()
    ctx := server.NewDefaultContext()
    cobra.EnableCommandSorting = false
    rootCmd := &cobra.Command{
        Use:               "nameshaked",
        Short:             "Nameshake App Daemon (server)",
        PersistentPreRunE: server.PersistentPreRunEFn(ctx),
    }

    server.AddCommands(ctx, cdc, rootCmd, appInit,
        server.ConstructAppCreator(newApp, "nameshake"),
        server.ConstructAppExporter(exportAppStateAndTMValidators, "nameshake"))

    // prepare and add flags
    executor := cli.PrepareBaseCmd(rootCmd, "NS", DefaultNodeHome)
    err := executor.Execute()
    if err != nil {
        // handle with #870
        panic(err)
    }
}

func newApp(logger log.Logger, db dbm.DB, traceStore io.Writer) abci.Application {
    return app.NewNameshakeApp(logger, db)
}

func exportAppStateAndTMValidators(
    logger log.Logger, db dbm.DB, traceStore io.Writer,
) (json.RawMessage, []tmtypes.GenesisValidator, error) {
    return nil, nil, nil
}
```

Notes on the above code:
- Most of the code combines the CLI commands from 1. Tendermint, 2. CosmosSDK, and 3. Nameservice module
- The rest of the code helps the application generate genesis state from the configuration

## Nameshakecli

Finish up by building the `nameservicecli` command:

> _*NOTE*_: Your application needs to import the code you just wrote. Here the import path is set to this repository (`github.com/sunnya97/sdk-nameservice-example`). If you are following along in your own repo you will need to change the import path to reflect that (`github.com/{{ .Username }}/{{ .Project.Repo }}`).

```go
package cli

import (
	"fmt"

	"github.com/spf13/cobra"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
)

type QueryResult struct {
	Value string         `json:"value"`
	Owner sdk.AccAddress `json:"owner"`
	Price sdk.Coins      `json:"price"`
}

// GetCmdResolveName queries information about a name
func GetCmdResolveName(queryRoute string, cdc *codec.Codec) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "resolve [name]",
		Short: "resolve name",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			name := args[0]
			cliCtx := context.NewCLIContext().WithCodec(cdc)

			res, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/proposals/%s", queryRoute, name), nil)
			if err != nil {
				fmt.Printf("could not resolve name - %s \n", string(name))
				return nil
			}

			fmt.Println(string(res))

			return nil
		},
	}

	return cmd
}

// GetCmdWhois queries information about a domain
func GetCmdWhois(queryRoute string, cdc *codec.Codec) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "whois [name]",
		Short: "Query whois info of name",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			name := args[0]
			cliCtx := context.NewCLIContext().WithCodec(cdc)

			res, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/whois/%s", queryRoute, name), nil)
			if err != nil {
				fmt.Printf("could not resolve whois - %s \n", string(name))
				return nil
			}

			fmt.Println(string(res))

			return nil
		},
	}

	return cmd
}
```

## Dependencies

### `Gopkg.toml`

Golang has a few dependency management tools. In this tutorial you will be using [`dep`](https://golang.github.io/dep/). `dep` uses a `Gopkg.toml` file in the root of the repository to define what dependencies the application needs. CosmosSDK apps currently depend on specific versions of some libraries. The below manifest contains all the necessary versions

```toml
# Gopkg.toml example
#
# Refer to https://github.com/golang/dep/blob/master/docs/Gopkg.toml.md
# for detailed Gopkg.toml documentation.
#
# required = ["github.com/user/thing/cmd/thing"]
# ignored = ["github.com/user/project/pkgX", "bitbucket.org/user/project/pkgA/pkgY"]
#
# [[constraint]]
#   name = "github.com/user/project"
#   version = "1.0.0"
#
# [[override]]
#   name = "github.com/x/y"
#   version = "2.4.0"
#
# [prune]
#   non-go = false
#   go-tests = true
#   unused-packages = true

[[constraint]]
  name = "github.com/cosmos/cosmos-sdk"
  branch = "develop"

[[override]]
  name = "github.com/golang/protobuf"
  version = "=1.1.0"

[[constraint]]
  name = "github.com/spf13/cobra"
  version = "~0.0.1"

[[constraint]]
  name = "github.com/spf13/viper"
  version = "~1.0.0"

[[override]]
  name = "github.com/tendermint/go-amino"
  version = "=v0.12.0"

[[override]]
  name = "github.com/tendermint/iavl"
  version = "=v0.11.0"

[[override]]
  name = "github.com/tendermint/tendermint"
  version = "=0.25.0"

[prune]
  go-tests = true
  unused-packages = true
```

### Makefile

Let us finish our application by writing the makefile.

```
get_tools:
ifdef DEP_CHECK
    @echo "Dep is already installed.  Run 'make update_tools' to update."
else
    @echo "Installing dep"
    go get -v $(DEP)
endif
ifdef STATIK_CHECK
    @echo "Statik is already installed.  Run 'make update_tools' to update."
else
    @echo "Installing statik"
    go version
    go get -v $(STATIK)
endif

get_vendor_deps:
    @echo "--> Generating vendor directory via dep ensure"
    @rm -rf .vendor-new
    @dep ensure -v -vendor-only

install:
    go install ./cmd/nameshaked
    go install ./cmd/nameshakecli
```

## Installing the software

First, you need to install `dep`. Below there is a command for using a shell script from `dep`'s site to preform this install. If you are uncomfortable `|`ing `curl` output to `sh` (you should be) then check out [your platform specific installation instructions](https://golang.github.io/dep/docs/installation.html).

```
go get -v github.com/golang/dep/cmd/dep
```

Next, run

```
dep ensure
```

Finally, run

```
make install
```

## Running the app

To initialize configuration and a `genesis.json` file for your application and an account for the transactions start by running:

> _*NOTE*_: Copy the `Address` output here and save it for later use

```bash
# Copy the chain_id output here and save it for later use
nameshaked init

# Copy the `Address` output here and save it for later use
nameshakecli keys add your_name
```

Next open the generated file `~/.nameshaked/config/genesis.json` in a text editor and copy in the address output by adding a key. This will give you control over a wallet with some coins when you start your local network. You can now start `nameshaked` by calling `nameshaked start`. You will see blocks being produced.

Open another terminal to run commands against the network you have just created:

> _*NOTE*_: In the below commands `--chain-id` and `accountaddr` are pulled using terminal utilities. You can also just input the raw strings saved from bootstrapping the network above. The commands require `jq` to be installed on your machine.

```bash
# First check the account to ensure you have funds
nameshakecli query account $(nameshakecli keys list -o json | jq -r .[0].address) --chain-id $(cat ~/.nameshaked/config/genesis.json | jq -r .chain_id)

# Buy your first name using your coins!
nameshakecli tx your_name.id 1mycoin --from $(nameshakecli keys list -o json | jq -r .[0].address) --chain-id $(cat ~/.nameshaked/config/genesis.json | jq -r .chain_id)

```
