﻿# getblockhash Method

Return the hash value of the corresponding block based on the specified index.

## Parameter Description

Index: Block index (block height)

## Example

Request body:

```json
{
   "jsonrpc": "2.0",
   "method": "getblockhash",
   "params": [10000],
   "id": 1
}
```

Response body:

```json
{
   "jsonrpc": "2.0",
   "id": 1,
   "result": "0x4c1e879872344349067c3b1a30781eeb4f9040d3795db7922f513f6f9660b9b2"
}
```