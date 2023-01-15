# The Basic Design of id builder

Author(s): tiltwind

Last updated: 2023-01-11

Discussion at https://github.com/idbuilder/proposal/issues/1.


## Abstract

This document is the basic design of `idbuilder`, includes basic types of id, which are auto-increment number id, snowflake style number id, and fixed format string id.

## Background

There is a lot of open source distribution id generate solutions, but I didn't find one which supports various of types of id, especially for the fixed format string id. `idbuilder` wants to be an open source solution to meet all requirements of id generation.

## Proposal


### Architecture

```
   ____________                            _______________                     _____________     
  |            |  ---- id requsest --->   |               |                   |             |
  | client/sdk |                          |     worker    | <---- config ---- |  controller |
  |____________|  <--- id response ----   |_______________|                   |_____________|
                                     
```

- **controller**: config the id generating rules, auth token
- **worker**: the id build work node
- **client/sdk**: send id request to the worker


### ID Types

The id builder should provide API to generate IDs of the following types:
- **auto-increment number id**
  - with delta 1: `1,2,3,4,...`
  - with delta 2: `1,3,5,7,9,...`
- **Snowflake style number id**
  - a number consist of `<timestamp> + <worker id> + <sequence>`
- **Fixed format string id**
  - the format `<type> + <YYYYMMDD> + '-' + <sequence>` will generate id like `P20230101-1234`, `B20230102-3456`.


### Auto-Increment Number Id

Parameters to config an increment number id builder:
- **base**: the number which to start from, it will increase when generating new.
- **delta**: the default delta between two ids. Using the request parameter if it exists.
- **max_request_delta**: the maximum limit value of the delta for the request delta, to avoid id increases too fast for invalid requests.
- **rand_delta**: the default value whether using random delta in the scope of the delta.


The client/sdk sends id requests for every id request.

```
                      key=a1234567, token=token123
   ____________     size=5, delta=5, rand_delta=true        _______________
  |            |  -------------- GET ------------------>   |               |
  | client/sdk |                                           |     worker    |
  |____________|  <------------ Response --------------    |_______________|
                         [5,8,12,13,16]
```

Request parameters:
- **key**: id key of the id config.
- **token**: the auth token, which is authorsied to access the id of the key.
- **size**: how many id requested, optional, default 1.
- **delta**: the delta between two ids, optional, default 1.
- **rand_delta**: whether use random delta, which is in the scope with the max value the parameter `delta` given. It's optional.

Request example:
```bash
curl http://idbuilder/v1/auto-increase?key=a1234567&token=token123&size=5&delta=5&rand_delta=true
```

Response structure:
- **id**: the id list generated

Response example:
```json
{
  "id": [5,8,12,13,16]
}
```


### Snowflake Style Number Id

Parameters to config a snowflake style number id builder:
- **skip_size**: the bit size to skip at the first beginning.
- **base_ts**: the base timestamp
- **ts_size**: the size of the timestamp bit part.
- **work_id_size**: the size of the work id bit part.
- **seq_size**: the size of the sequence bit part.


The client/sdk requests snowflake style config when initializing, and then it will generate the id itself.

```
  _____________       key=a1234567, token=token123          _______________
  |            |  -------------- GET ------------------>   |               |
  | client/sdk |                                           |     worker    |
  |____________|  <------------ Response --------------    |_______________|
                          snowflake id config
```

Request parameters:
- **key**: id key of the id config.
- **token**: the auth token, which is authorized to access the id of the key.

Request example:
```bash
curl http://idbuilder/v1/snowflake?key=a1234567&token=token123
```

Response structure:
- **skip_size**: the bit size to skip at the first beginning.
- **base_ts**: the base timestamp
- **ts_size**: the size of the timestamp bit part.
- **work_id**: the assigned work id for the client.
- **work_id_size**: the size of the work id bit part.
- **seq_size**: the size of the sequence bit part.

Response example:
```json
{
  "skip_size":1,
  "base_ts": 1673606841,
  "ts_size": 41,
  "work_id": 5,
  "work_id_size": 10,
  "seq_size": 12
}
```

The client/sdk will generate IDs using the config:
```
           0  0000000 00000000 00000000 00000000 00000000 00 000000 0000 0000 00000000
< fix zero >  <          current_ts - base_ts              > < work_id > <   seqno   >
   < 1 bit >  <--------------- 41 bits --------------------> < 10 bits > <- 12 bits ->
```


### Fixed Format String Id

Parameters to config a snowflake style number id builder:
- **parts**: one or multiple `part_config`, the id value that combines all part values by order.

There are various types of `part_config` : 

- **fixed-chars**: the fixed character(s)
  - **fix-chars-value**: config the value of the fix chars.

- **fixed-polling-char**: the fixed character in polling mode from a given scope of characters.
  - **chars-scope**: a string including all chars for polling select.

- **fixed-random-chars**: the fixed characters in random mode from a given scope of characters.
  - **chars-scope**: a string including all chars for random selection.
  - **length**: the length of the value.

- **date-format**: the string format date value of the current time.
  - **format**: the date format, eg, `yyyyMMddhhmmssSSS`.
  - **time_zone**: the time zone of the date.

- **timestamp**: the milliseconds of the current time.
  - **base_ts**: the timestamp to subtract.

- **unix-seconds**: the Unix seconds of the current time
  - **base_unix**: the base Unix seconds to subtract.

- **auto-increase**: refer to an auto-increment number id config
  - **length-fixed**: whether padding the value to fix length when not enough.
  - **length**: the length of the value.
  - **padding-mode**: adding `prefix`/`suffix` to the value.
  - **padding-char**: the padding character.
  - **number-base**: the base to format number, the valid value is from 2 to 36.
  - **reset_scope**: when to reset the number.
    - **none**: not to reset the number.
    - **year**: the year of the current time.
    - **month**: the month of the current time. 
    - **date**: the date of the current time.


The client/sdk requests fixed format string IDs for every id request.

```
                      key=a1234567, token=token123
   ____________                  size=2                     _______________
  |            |  -------------- GET ------------------>   |               |
  | client/sdk |                                           |     worker    |
  |____________|  <------------ Response --------------    |_______________|
                         ["P2301-1234","P2301-1235"]
```

Request parameters:
- **key**: id key of the id config.
- **token**: the auth token, which is authorized to access the id of the key.
- **size**: how many id requested, optional, default 1.


Request example:
```bash
curl http://idbuilder/v1/snowflake?key=a1234567&token=token123&size=2
```

Response structure:
- **id**: the id list generated

Response example:
```json
{
  "id": ["P2301-1234","P2301-1235"]
}
```
