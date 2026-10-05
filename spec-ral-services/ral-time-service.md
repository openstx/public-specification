# RAL Time Service

## Introduction

This service defines primitives that provide access to the RAL's local timestamp. The `RAL_Time_Instant_Now_Command` can be used to obtain the current timestamp represented as `RalInstant`. The caller can use `RAL_Time_Instant_Diff_Command` to compute the difference between two timestamps. 

### Related Documents

| Document                          | Description                  |
| --------------------------------- | ---------------------------- |
| [Introduction](./introduction.md) | Introduction to the RAL |
| [RAL Overview](./ral-overview.md) | Overview of the RAL's main concepts |

## Commands

<a name="ral-instant-service-RAL_Time_Instant_Now_Command"></a>

### `RAL_Time_Instant_Now_Command`

**Summary**: Requests the current RAL timestamp.

**Parameters**: None.

**Event Pattern**: (`RAL_Time_Instant_Now_Command -> RAL_Time_Instant_Now_CommandEndEvent`)

**Description**:

The implementation MUST return the radio's current `RalInstant`, as defined in in [§RalInstant](ral-overview.md#ralinstant). 

The implementation MUST _immediately_ capture the `RalInstant` and generate the solicited `RAL_Time_Instant_Now_CommandEndEvent`, without queuing or deferred processing.

If the data type used for `RalInstant` is wider than `RAL_TICK_COUNTER_WIDTH` bits, the implementation MUST ensure that the higher bits above position `RAL_TICK_COUNTER_WIDTH - 1` are set to `0`. 

<!-- ((tbd)) define RAL_TICK_MIN_WRAPAROUND_PERIOD_<UNIT>, including the selection of the appropriate <UNIT>, ass soon as higher-layer OpenSTX requirements are more precise -->
The RAL MUST ensure that the wrap-around period `P = 2^RAL_TICK_COUNTER_WIDTH / RAL_TICK_FREQ_HZ` is at least `RAL_TICK_MIN_WRAPAROUND_PERIOD_<UNIT>`.


<a name="ral-instant-service-RAL_Time_Instant_Diff_Command"></a>

### `RAL_Time_Instant_Diff_Command`

**Summary**: Requests the signed tick-count difference between two RAL timestamps, enabling both elapsed-time measurement and ordering.

**Parameters**:

| Name | Type         | Valid Range                   | Description                                                    |
| ---- | ------------ | ----------------------------- | -------------------------------------------------------------- |
| `t1` | `RalInstant` | As defined in `RAL_Time_Instant_Now_Command`. | First RAL instant. |
| `t2` | `RalInstant` | As defined in `RAL_Time_Instant_Now_Command`. | Second RAL instant. |

**Event Pattern**: (`RAL_Time_Instant_Diff_Command -> RAL_Time_Instant_Diff_CommandEndEvent`)

**Description**:

The implementation MUST compute the difference `diff = t2 - t1` represented as `RalInstantDiff`, as defined in [§RalInstantDiff](ral-overview.md#ralinstantdiff)

If the data type used for `t1` and `t2` is wider than `RAL_TICK_COUNTER_WIDTH` bits, the implementation MUST discard the bits above position `RAL_TICK_COUNTER_WIDTH - 1` before computing the result. 

The command assumes that the actual, absolute time difference between capturing `t1` and `t2` is less than `P / 2`, where `P` is the wrap-around period defined as `P = 2^RAL_TICK_COUNTER_WIDTH / RAL_TICK_FREQ_HZ`. If that is not the case, the returned result is undefined and the caller MUST NOT rely on its value.

The implementation MUST produce the same `diff` as the following procedure.

```
1. BEGIN PROCEDURE compute_diff
2. Let sign_extend(x, n) be the sign-extension of x by n bits. 
3. Let DT_WIDTH >= RAL_TICK_COUNTER_WIDTH be the bit-width of the datatype used to represent t1 and t2.  
4. Compute u_diff := (t2 - t1) mod 2^RAL_TICK_COUNTER_WIDTH in the domain of RalInstant
5. Map u_diff to the domain of RalInstantDiff by computing diff := sign_extend(u_diff, DT_WIDTH - RAL_TICK_COUNTER_WIDTH) 
6. END PROCEDURE compute_diff
```

The caller can interpret the resulting value of `diff` as follows, independent of a potential wrap-around of the tick counter between capturing `t1` and `t2`:
* a positive `diff` indicates that `t1` happened before `t2`,
* a negative `diff` indicates that `t1` happened after `t2`,
* a `diff` of `0` indicates that `t1` and `t2` correspond to the same point in time.

The following examples 1-4 show how the procedure `compute_diff` is applied in different scenarios, and how the resulting `diff` is interpreted.

Example 1. Execution of `compute_diff` given a 16-bit counter, `t1 = 0x0015` , `t2 = 0x0A39`. Expectation: `t1` is `0x0A24` before `t2`.

```
u_diff = (0x0A39 - 0x0015) mod 2^16 = 0x0A24
diff = sign_extend(0x0A24, 0) = 0x0A24 {1}
(diff as RalInstantDiff) > 0 {2}

{1} ^ {2} => "t1 happened 0x0A24 before t2"
```

Example 2. Execution of `compute_diff` given a 16-bit counter, `t1 = 0xFF47` , `t2 = 0x0021`. Expectation: `t1` is `0x00DA` before `t2`.

```
u_diff = (0x0021 - 0xFF47) mod 2^16 = 0x00DA
diff = sign_extend(0x00DA, 0) = 0x00DA {1}
(diff as RalInstantDiff) > 0 {2}

{1} ^ {2} => "t1 happened 0x00DA before t2"
```

Example 3. Execution of `compute_diff` given a 16-bit counter, `t1 = 0x0015` , `t2 = 0xFE12`. Expectation: `t1` is `0x0203` after `t2`.

```
u_diff = (0xFE12 - 0x0015) mod 2^16 = 0xFDFD
diff = sign_extend(0xFDFD, 0) = 0xFDFD {1}
(diff as RalInstantDiff) < 0 {2} 

{1} ^ {2} => "t1 happened -0xFDFD = 0x0203 after t2"
```

Example 4. Execution of `compute_diff` given a 10-bit counter, `t1 = 0x0015` , `t2 = 0x03F1`. The 10-bit input values are passed as 16-bit unsigned integers. Expectation: `t1` is `0x0024` after `t2`.

```
u_diff = (0x03F1 - 0x0015) mod 2^10 = 0x03DC
diff = sign_extend(0x03DC, 6) = 0xFFDC {1}
(diff as RalInstantDiff) < 0 {2}

{1} ^ {2} => "t1 happened -0xFFDC = 0x0024 after t2"
```

## Events

<a name="ral-instant-service-RAL_Time_Instant_Now_CommandEndEvent"></a>

### `RAL_Time_Instant_Now_CommandEndEvent`

**Summary**: Reports on the result of the command requesting the current `RalInstant`.

**Soliciting Command**: `RAL_Time_Instant_Now_Command`

**Parameters**:

| Name      | Type         | Valid Range                              | Description                                                                                               |
| --------- | ------------ | ---------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `command_status`  | Enumeration  | `SUCCESS`, `FAILURE`                     | Outcome of the command. If the requested `RalInstant` can not be obtained, the `command_status` MUST be `FAILURE` |
| `instant` | `RalInstant` | As defined in [§RalInstant](ral-overview.md#ralinstant). | The requested RAL instant. If `command_status` is `SUCCESS`, then `instant` MUST be present.                                       |

**Description**: None.

<a name="ral-instant-service-RAL_Time_Instant_Diff_CommandEndEvent"></a>

### `RAL_Time_Instant_Diff_CommandEndEvent`

**Summary**: Reports on the result of the command computing the difference between two RAL timestamps.

**Soliciting Command**: `RAL_Time_Instant_Diff_Command`

**Parameters**:

| Name     | Type             | Valid Range                                            | Description                                                                                         |
| -------- | ---------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| `diff`   | `RalInstantDiff` | As defined in [§RalInstantDiff](ral-overview.md#ralinstantdiff). | The computed difference between two RAL instants.|

**Description**: None.

