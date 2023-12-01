<details>
  
  <summary>
    
    Source Code
    
  </summary>
  
  
  ```Rust
/*
* Copyright (C) 2019-2021 TON Labs. All Rights Reserved.
*
* Licensed under the SOFTWARE EVALUATION License (the "License"); you may not use
* this file except in compliance with the License.
*
* Unless required by applicable law or agreed to in writing, software
* distributed under the License is distributed on an "AS IS" BASIS,
* WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
* See the License for the specific TON DEV software governing permissions and
* limitations under the License.
*/

//! Function and event param types.

use crate::{AbiError, Param};
use crate::contract::{AbiVersion, ABI_VERSION_1_0, ABI_VERSION_2_0, ABI_VERSION_2_1, ABI_VERSION_2_4};
use std::fmt;

use ton_types::{error, Result};

/// Function and event param types.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ParamType {
    /// uint<M>: unsigned integer type of M bits.
    Uint(usize),
    /// int<M>: signed integer type of M bits.
    Int(usize),
    /// varuint<M>: variable length unsigned integer type of maximum M bytes.
    VarUint(usize),
    /// varint<M>: variable length integer type of maximum M bytes.
    VarInt(usize),
    /// bool: boolean value.
    Bool,
    /// Tuple: several values combined into tuple.
    Tuple(Vec<Param>),
    /// T[]: dynamic array of elements of the type T.
    Array(Box<ParamType>),
    /// T[k]: dynamic array of elements of the type T.
    FixedArray(Box<ParamType>, usize),
    /// cell - tree of cells
    Cell,
    /// hashmap - values dictionary
    Map(Box<ParamType>, Box<ParamType>),
    /// TON message address
    Address,
    /// byte array
    Bytes,
    /// fixed size byte array
    FixedBytes(usize),
    /// UTF8 string
    String,
    /// Nanograms
    Token,
    /// Timestamp
    Time,
    /// Message expiration time
    Expire,
    /// Public key
    PublicKey,
    /// Optional parameter
    Optional(Box<ParamType>),
    /// Parameter stored in reference
    Ref(Box<ParamType>),
}

impl fmt::Display for ParamType {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.type_signature())
    }
}

impl ParamType {
    /// Returns type signature according to ABI specification
    pub fn type_signature(&self) -> String {
        match self {
            ParamType::Uint(size) => format!("uint{}", size),
            ParamType::Int(size) => format!("int{}", size),
            ParamType::VarUint(size) => format!("varuint{}", size),
            ParamType::VarInt(size) => format!("varint{}", size),
            ParamType::Bool => "bool".to_owned(),
            ParamType::Tuple(params) => {
                let mut signature = "".to_owned();
                for param in params {
                    signature += ",";
                    signature += &param.kind.type_signature();
                }
                signature.replace_range(..1, "(");
                signature + ")"
            }
            ParamType::Array(ref param_type) => format!("{}[]", param_type.type_signature()),
            ParamType::FixedArray(ref param_type, size) => {
                format!("{}[{}]", param_type.type_signature(), size)
            }
            ParamType::Cell => "cell".to_owned(),
            ParamType::Map(key_type, value_type) => format!(
                "map({},{})",
                key_type.type_signature(),
                value_type.type_signature()
            ),
            ParamType::Address => format!("address"),
            ParamType::Bytes => format!("bytes"),
            ParamType::FixedBytes(size) => format!("fixedbytes{}", size),
            ParamType::String => format!("string"),
            ParamType::Token => format!("gram"),
            ParamType::Time => format!("time"),
            ParamType::Expire => format!("expire"),
            ParamType::PublicKey => format!("pubkey"),
            ParamType::Optional(ref param_type) => {
                format!("optional({})", param_type.type_signature())
            }
            ParamType::Ref(ref param_type) => format!("ref({})", param_type.type_signature()),
        }
    }

    pub fn set_components(&mut self, components: Vec<Param>) -> Result<()> {
        match self {
            ParamType::Tuple(params) => {
                if components.len() == 0 {
                    Err(error!(AbiError::EmptyComponents))
                } else {
                    Ok(*params = components)
                }
            }
            ParamType::Array(array_type) => array_type.set_components(components),
            ParamType::FixedArray(array_type, _) => array_type.set_components(components),
            ParamType::Map(_, value_type) => value_type.set_components(components),
            ParamType::Optional(inner_type) => inner_type.set_components(components),
            ParamType::Ref(inner_type) => inner_type.set_components(components),
            _ => {
                if components.len() != 0 {
                    Err(error!(AbiError::UnusedComponents))
                } else {
                    Ok(())
                }
            }
        }
    }

    /// Check if parameter type is supoorted in particular ABI version
    pub fn is_supported(&self, abi_version: &AbiVersion) -> bool {
        match self {
            ParamType::Time | ParamType::Expire | ParamType::PublicKey => {
                abi_version >= &ABI_VERSION_2_0
            }
            ParamType::String
            | ParamType::Optional(_)
            | ParamType::VarInt(_)
            | ParamType::VarUint(_) => abi_version >= &ABI_VERSION_2_1,
            ParamType::Ref(_) => abi_version >= &ABI_VERSION_2_4,
            _ => abi_version >= &ABI_VERSION_1_0,
        }
    }
}


  ```
</details>
This Rust code appears to be a part of a serialization and deserialization implementation for the `ParamType` enum in the context of ABI (Application Binary Interface) handling, likely related to blockchain and smart contract development. Let's break down the key components and functionalities:

1. **Deserialization Implementation for ParamType:**
   - The `Deserialize` trait is implemented for the `ParamType` enum.
   - The `deserialize` function uses a custom visitor, `ParamTypeVisitor`, to handle deserialization.
   - The `visit_str` and `visit_string` functions of the visitor handle deserialization from a string, calling the `read_type` function.

2. **ParamTypeVisitor:**
   - This struct implements the `Visitor` trait for deserializing `ParamType`.
   - It defines expectations for the deserializer and implements methods for visiting a string, converting it to a `ParamType` by calling the `read_type` function.

3. **read_type Function:**
   - This function converts a string representation of a parameter type into a `ParamType` enum.
   - It handles various types, including bool, tuple, int, uint, varint, varuint, map, cell, address, token, bytes, fixedbytes, time, expire, pubkey, string, optional, and ref.
   - For certain types like arrays (`[]`), it distinguishes between fixed and dynamic arrays.

4. **Error Handling:**
   - The code includes error handling for invalid names or failed conversions, raising `AbiError` in case of issues.

5. **Dependencies:**
   - The code relies on external dependencies such as `serde` for serialization/deserialization and `ton_types` for Result and error handling.

6. **Rust-Specific Features:**
   - The code utilizes Rust's pattern matching, ownership system, and error handling mechanisms.

7. **ABI Handling:**
   - The code specifically deals with ABI-related concepts, parsing parameter types commonly used in smart contract ABI.

8. **Map Handling:**
   - There's special handling for the "map" type, ensuring that only specific types can be used as keys.

9. **Optional and Ref Types:**
   - The code supports optional and reference types in the ABI.

10. **Numeric Type Parsing:**
   - The code extracts numeric values from type names for integer types.

11. **FixedBytes Type:**
   - The code handles the "fixedbytes" type with a specified length.

This code seems well-structured and covers a comprehensive set of parameter types relevant to ABI in blockchain and smart contract development. If you have specific questions or if there's anything specific you'd like to explore in more detail, feel free to ask!
