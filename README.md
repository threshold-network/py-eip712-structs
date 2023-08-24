# EIP-712 Structs  [![Build Status](https://travis-ci.org/ConsenSys/py-eip712-structs.svg?branch=master)](https://travis-ci.org/ConsenSys/py-eip712-structs) [![Coverage Status](https://coveralls.io/repos/github/ConsenSys/py-eip712-structs/badge.svg?branch=master)](https://coveralls.io/github/ConsenSys/py-eip712-structs?branch=master)

A python interface for simple EIP-712 struct construction.

In this module, a "struct" is structured data as defined in the standard.
It is not the same as the Python Standard Library's struct (e.g., `import struct`).

Read the proposal:<br/>
https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md

#### Supported Python Versions
- `3.9`
- `3.10`
- `3.11`

## Install
```bash
pip install eip712-structs
```

## Usage
See [API.md](API.md) for a succinct summary of available methods.

Examples/Details below.

#### Quickstart
Say we want to represent the following struct, convert it to a message and sign it:
```text
struct MyStruct {
    string some_string;
    uint256 some_number;
}
```

With this module, that would look like:
```python
# Make a unique domain
from eip712_structs import make_domain
domain = make_domain(name='Some name', version='1.0.0')  # Make a Domain Separator

# Define your struct type
from eip712_structs import EIP712Struct, String, Uint
class MyStruct(EIP712Struct):
    some_string = String()
    some_number = Uint(256)

# Create an instance with some data
mine = MyStruct(some_string='hello world', some_number=1234)

# Values can be get/set dictionary-style:
mine['some_number'] = 4567
assert mine['some_string'] == 'hello world'
assert mine['some_number'] == 4567

# Into a message dict - domain required
my_msg = mine.to_message(domain)

# Into message JSON - domain required.
# This method converts bytes types for you, which the default JSON encoder won't handle.
my_msg_json = mine.to_message_json(domain)

# Into signable bytes - domain required
my_bytes = mine.signable_bytes(domain)
```

See [Member Types](#member-types) for more information on supported types.

#### Dynamic construction
Attributes may be added dynamically as well. This may be necessary if you
want to use a reserved keyword like `from`.

```python
from eip712_structs import EIP712Struct, Address
class Message(EIP712Struct):
    pass

Message.to = Address()
setattr(Message, 'from', Address())

# At this point, Message is equivalent to `struct Message { address to; address from; }`

```

#### The domain separator
EIP-712 specifies a domain struct, to differentiate between identical structs that may be unrelated.
A helper method exists for this purpose.
All values to the `make_domain()`
function are optional - but at least one must be defined. If omitted, the resulting
domain struct's definition leaves out the parameter entirely.

The full signature: <br/>
`make_domain(name: string, version: string, chainId: uint256, verifyingContract: address, salt: bytes32)`

##### Setting a default domain
Constantly providing the same domain can be cumbersome. You can optionally set a default, and then forget it.
It is automatically used by `.to_message()` and `.signable_bytes()`

```python
import eip712_structs

foo = SomeStruct()

my_domain = eip712_structs.make_domain(name='hello world')
eip712_structs.default_domain = my_domain

assert foo.to_message() == foo.to_message(my_domain)
assert foo.signable_bytes() == foo.signable_bytes(my_domain)
```

## Member Types

### Basic types
EIP712's basic types map directly to solidity types.

```python
from eip712_structs import Address, Boolean, Bytes, Int, String, Uint

Address()  # Solidity's 'address'
Boolean()  # 'bool'
Bytes()    # 'bytes'
Bytes(N)   # 'bytesN' - N must be an int from 1 through 32
Int(N)     # 'intN' - N must be a multiple of 8, from 8 to 256
String()   # 'string'
Uint(N)    # 'uintN' - N must be a multiple of 8, from 8 to 256
```

Use like:
```python
from eip712_structs import EIP712Struct, Address, Bytes

class Foo(EIP712Struct):
    member_name_0 = Address()
    member_name_1 = Bytes(5)
    # ...etc
```

### Struct references
In addition to holding basic types, EIP712 structs may also hold other structs!
Usage is almost the same - the difference is you don't "instantiate" the class.

Example:
```python
from eip712_structs import EIP712Struct, String

class Dog(EIP712Struct):
    name = String()
    breed = String()

class Person(EIP712Struct):
    name = String()
    dog = Dog  # Take note - no parentheses!

# Dog "stands alone"
Dog.encode_type()     # Dog(string name,string breed)

# But Person knows how to include Dog
Person.encode_type()  # Person(string name,Dog dog)Dog(string name,string breed)
```

Instantiating the structs with nested values may be done a couple different ways:

```python
# Method one: set it to a struct
dog = Dog(name='Mochi', breed='Corgi')
person = Person(name='E.M.', dog=dog)

# Method two: set it to a dict - the underlying struct is built for you
person = Person(
    name='E.M.',
    dog={
        'name': 'Mochi',
        'breed': 'Corgi',
    }
)
```

### Arrays
Arrays are also supported for the standard.

```python
array_member = Array(<item_type>[, <optional_length>])
```

- `<item_type>` - The basic type or struct that will live in the array
- `<optional_length>` - If given, the array is set to that length.

For example:
```python
dynamic_array = Array(String())      # String[] dynamic_array
static_array  = Array(String(), 10)  # String[10] static_array
struct_array = Array(MyStruct, 10)   # MyStruct[10] - again, don't instantiate structs like the basic types
```

## Development
Contributions always welcome.

Install dependencies:
- `pip install -r requirements.txt`

Run tests:
- `python setup.py test`
- Some tests expect an active local ganache chain on http://localhost:8545. Docker will compile the contracts and start the chain for you.
- Docker is optional, but useful to test the whole suite. If no chain is detected, chain tests are skipped.
- Usage:
    - `docker-compose up -d` (Starts containers in the background)
    - Note: Contracts are compiled when you run `up`, but won't be deployed until the test is run.
    - Cleanup containers when you're done: `docker-compose down`

Deploying a new version:
- Bump the version number in `setup.py`, commit it into master.
- Make a release tag on the master branch in Github. Travis should handle the rest.


## Shameless Plug
Written by [ConsenSys](https://consensys.net) for the world! :heart:

## Historical Note

Maintenance of this project was taken over by Threhsold Network in August 2023.

Five years prior, in August 2018, NuCypher, which later joined Threhsold Network, launched a gitcoin bounty for the adoption and implementation of EIP-712 with help from Scott Moore and Piper Merriam, and became the first project to use an EIP-712 implementation in production. So it is a proud full-circle moment that we bring this project back under our umbrella and make it compatible with current versions of Python.

