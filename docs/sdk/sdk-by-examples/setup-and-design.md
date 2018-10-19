# Setup And Design

## Prerequisites

- Have [go](https://golang.org/dl/), [git](https://git-scm.com/downloads) installed
- Don't forget to set your `PATH` and `GOPATH`

## Application design principles

When designing a module, it is good to adopt a certain methodology. Remember that a blockchain application is just a replicated state-machine. The state is the representation of the application at a given time. It is up to the application developer to define what the state represents, depending on the goal of the application. For example, the state of a simple cryptocurrency application will be a mapping of addresses to balances.

The state can be updated according to predefined rules. Given a state and a transaction, the state-machine (i.e. the application) will return a new state. In a blockchain application, transactions are bundled in blocks, but the logic is the same. Given a state and a set of transactions (a block), the application returns a new state. A SDK module is just a subset of the application, but it is based on the same principles. As a result, module developers only have to define a subset of the state and a subset of the transaction types, which trigger state transitions.

+-----------+      +------------------+      +-----------+
|           |      |                  |      |           |
|  State 1  +------>   Block of Txs   +------>  State 2  |
|           |      |                  |      |           |
+-----------+      +------------------+      +-----------+

A good first step in designing an application is to define a simplified specification of your application.
From this specification, you can then define the `state` and `messages`. 

