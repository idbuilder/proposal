# The Basic Design of id builder

Author(s): tiltwind

Last updated: 2023-01-11

Discussion at https://github.com/idbuilder/proposal/issues/1.


## Abstract

This document is the basic design of id builder, includes basic types of id, which are auto increment number id, snowflaw style number id, and fix format string id.

## Background

There are a lot of open source distribution id generate solutions, but I didn't find one which supports variaty types of id, especially for the fix format string id. `idbuilder` wants to be a open source solution to meet all reqirements of id generation.

## Proposal


### Architeture

```
   ____________                            _______________                     _____________     
  |            |  ---- id requsest --->   |               |                   |             |
  | client/sdk |                          |     worker    | <---- config ---- |  controller |
  |____________|  <--- id response ----   |_______________|                   |_____________|
                                     
```

- **controller**: config the id generating rules, auth token
- **worker**: the id build work node
- **client/sdk**: send id request to the worker


### API Design

The id builder should provide api to generate id of the following types:
- **Auto increment number id**
  - with delta 1: `1,2,3,4,...`
  - with delta 2: `1,3,5,7,9,...`
- **Snowflake style number id**
  - a number consist of `<timestamp> + <worker id> + <sequence>`
- **Fix format string id**
  - the format `<type> + <YYYYMMDD> + '-' + <sequence>` will generate id like `P20230101-1234`, `B20230102-3456`.


#### auto increment number id

Parameters to config a increment number id builder:
- **base**: number which to start from, it will increase when generating new.
- **delta**: the default delta between two id. Using the request parameter if exists.
- **max_request_delta**: the maxinum limit value of the delta for the request delta, to avoid id increases too fast for invalid request.
- **rand_delta**: the default value whether using random delta in the scope of delta.


The client/sdk send id request for every id request.

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
- **token**: the auth token, which is authoried to access the id of the key.
- **size**: how many id requested, optional, default 1.
- **delta**: the delta between two id, optional, default 1.
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


#### snowflake style number id

Parameters to config a snowflake style number id builder:
- **skip_size**: the bit size to skip at the first beginning.
- **base_ts**: the base timestamp
- **ts_size**: the size of the timestamp bit part.
- **work_id_size**: the size of the work id bit part.
- **seq_size**: the size of the sequence bit part.


The client/sdk request snowflake style config when initinizing, and then it will generate id itself.

```
  _____________       key=a1234567, token=token123          _______________
  |            |  -------------- GET ------------------>   |               |
  | client/sdk |                                           |     worker    |
  |____________|  <------------ Response --------------    |_______________|
                          snowflake id config
```

Request parameters:
- **key**: id key of the id config.
- **token**: the auth token, which is authoried to access the id of the key.

Request example:
```bash
curl http://idbuilder/v1/snowflake?key=a1234567&token=token123
```

Response structure:
- **skip_size**: the bit size to skip at the first beginning.
- **base_ts**: the base timestamp
- **ts_size**: the size of the timestamp bit part.
- **work_id**: the assigned work id for client.
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

The client/sdk will generate id using the config:
```
           0  0000000 00000000 00000000 00000000 00000000 00 000000 0000 0000 00000000
< fix zero >  <          current_ts - base_ts              > < work_id > <   seqno   >
   < 1 bit >  <--------------- 41 bits --------------------> < 10 bits > <- 12 bits ->
```


#### fix format string id

Parameters to config a snowflake style number id builder:
- **parts**: one or multiple `part_config`, the id value is that combines all part values by order.

There are variouse types of `part_config` : 

- **fix-chars**: fix character(s)
  - **fix-chars-value**: config the value of the fix chars.

- **fix-polling-char**: fix character in polling mode from a given scope of characters.
  - **chars-scope**: a string including all chars for polling select.

- **fix-random-chars**: fix characters in random mode from a given scope of characters.
  - **chars-scope**: a string including all chars for random select.
  - **length**: the length of the value.

- **date-format**: the string format date value of current time.
  - **format**: the date format, eg `yyyyMMddhhmmssSSS`.
  - **time_zone**: the time zone of the date.

- **timestamp**: the milliseconds of current time.
  - **base_ts**: the timestamp to substract.

- **unix-seconds**: the unix seconds of current time
  - **base_unix**: the base unix secones to substract.

- **auto-increase**: refer to a auto increment number id config
  - **length-fixed**: whether padding the value to fix length when not enough.
  - **length**: the length of the value.
  - **padding-mode**: adding `prefix`/`suffix` to the value.
  - **padding-char**: the padding character.
  - **number-base**: the base to format number, the valid value is from 2 to 36.
  - **reset_scope**: when to reset the number.
    - **none**: not to reset the number.
    - **year**: the year of current time.
    - **month**: the month of current time. 
    - **date**: the date of current time.


The client/sdk request fix format string id for every id request.

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
- **token**: the auth token, which is authoried to access the id of the key.
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
