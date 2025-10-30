# Static Analysis Tool for Solidity Smart Contracts

## Background

This fork of Solidity Compiler traverses Solidity AST and serializes nodes to JSON while preserving annotations such as types and referencedDeclaration ids. 

The analysis first collects state-variable and function declarations, then inspects expressions and statements (Identifiers, MemberAccess, IndexAccess, Assignments, FunctionCalls) to classify occurrences as reads or writes by syntactic position (RHS vs LHS) and contextual cues.

To handle storage aliases it tracks storage-reference variables and resolves referencedDeclaration ids to concrete state variables, and it aggregates per-function read/write sets while accounting for modifiers, external calls, and merged execution sequences.

The pipeline finally filters duplicates and likely false positives and emits a contract-level JSON summary of read/write and external read/write sets.

## Build and Install

Instructions about how to build and install the Solidity compiler can be
found in the [Solidity documentation](https://docs.soliditylang.org/en/latest/installing-solidity.html#building-from-source).

Need to create rwSet directoty for .json output.
```
cd ast-static-analysis
mkdir reSet
``` 

## Example

A hotel booking program in Solidity:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

contract HotelBooking {
  uint256 public constant MAX_ROOMS = 5;
  uint256 public constant HOTEL_PRICE = 20000;
  address public immutable paymentService;
  uint256 public roomReserved;
  address[MAX_ROOMS] public rooms;

  constructor(address _paymentService) {
    paymentService = _paymentService;
  }

  function checkRoomAvailability(address account, uint payment) public view returns (bool roomAvailability) {
    (bool success, bytes memory data) = paymentService.staticcall(abi.encodeWithSignature("verifyDeposit(address,uint256)", account, payment));
    require(success, "Payment Check Failed");

    bool depositAvailable = abi.decode(data, (bool));
    roomAvailability = (roomReserved < MAX_ROOMS && payment >= HOTEL_PRICE && depositAvailable == true);
  }

  function bookHotel(address account) public {
    (bool success, ) = paymentService.call(abi.encodeWithSignature("receivePayment(address,uint256)", account, HOTEL_PRICE));
    require(success, "Payment Failed");
    rooms[roomReserved++] = account;
  }
}
```

Output of read/write set ststic analysis for HotelBooking smart contract:
```
4 MAX_ROOMS
7 HOTEL_PRICE
9 paymentService
11 roomReserved
15 rooms
function: 4 uses global variable: MAX_ROOMS
function: constructor uses global variable: paymentService
function: checkRoomAvailability uses global variable: paymentService roomReserved MAX_ROOMS HOTEL_PRICE
function: bookHotel uses global variable: HOTEL_PRICE paymentService rooms roomReserved
```
```json
[
	{
		"externalRweSet" : [],
		"function" : "(address)",
		"functionSelector" : "",
		"readSet" : [],
		"writeSet" : []
	},
	{
		"externalRweSet" : 
		[
			{
				"name" : "verifyDeposit(address,uint256)",
				"type" : "execute"
			}
		],
		"function" : "checkRoomAvailability(address,uint256)",
		"functionSelector" : "fa864572",
		"readSet" : 
		[
			"roomReserved"
		],
		"writeSet" : []
	},
	{
		"externalRweSet" : 
		[
			{
				"name" : "receivePayment(address,uint256)",
				"type" : "execute"
			}
		],
		"function" : "bookHotel(address)",
		"functionSelector" : "165fcb2d",
		"readSet" : [],
		"writeSet" : 
		[
			"roomReserved",
			"rooms"
		]
	}
]
```

## Documentation

The Solidity documentation is hosted using [Read the Docs](https://docs.soliditylang.org).