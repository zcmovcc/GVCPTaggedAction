# GVCP Tagged Action

**Request For Comments**

* **Date**: 2023-04-03
* **Version**: 0.1.0
* **Author**: WenHua Zheng
* **Email**: z at cmov dot cc
* **Latest**: https://github.com/zcmovcc/GVCPTaggedAction

## 1. Abstract

This is a proposal to add *Tagged Action Commands* to GigE Vision standard and to add tag related features to GenICam Standards.

## 2. Introduction

TODO

## 3. Proposed Wording

### 3.1 Revision to the GigE Vision specification

These changes are based on: *GigE Vision, Video Streaming and Device Control Over Ethernet Standard, version 2.2*

#### 3.1.1 Changes to §14.3 Action Commands

* Insert the following chapter between **§14.3.1 Scheduled Action Commands** and **§14.3.2 ACTION_CMD examples**

> #### 14.3.2 Tagged Action Commands
> 
> Action commands may carry additional data within the command packet, called *action tag*.  The tag can contain arbitrary data. The format and semantics of the tag content are defined by the user.
> 
> One use case of action tag is to embed a piece of the tag into the stream payload, in the form of a chunk. This provides an information delivery channel from the action command emitter to the payload receiver, without explicit data connection between them. For example, a GUID can be used to uniquely identify each acquisition, without using unreliable counting mechanism or relatively complicated timestamp mechanism. Another example is using a type code to tell the receiver to treat the payload in different ways.
> 
> Another possible use of action tag is to dynamicly select different acquisition setting groups, such as exposure time/gain/.etc, based on the tag content.
> 
> There are 3 support levels for Tagged Action Commands:
> * **Level 0**: The bit 16 (*tagged_action_command*) of the `GVCP Capability` register is **0**. The device has no tag related function, and may or may not accept action commands longer than 20(unscheduled)/28(scheduled) bytes in length. This is for legacy devices developed without the concept of Tagged Action Commands in mind.
> * **Level 1**: The bit 16 (*tagged_action_command*) of the `GVCP Capability` register is **1**. The device recognizes action commands longer than 20/28 bytes, but ignores any tag in the command and consumes the command as if no tag is present. The device has no tag related function but can be used together with level-2 devices. In this case, level-1 and level-2 devices are able to react to the same broadcasted Tagged Action Command concurrently.
> * **Level 2**: The bit 16 (*tagged_action_command*) of the `GVCP Capability` register is **1**. The device supports Tagged Action Commands, and consumes the tags according to device configurations.
>  
> All the 3 levels are compliant to this standard. But new devices are recommended to support at least level 1.

#### 3.1.2 Changes to §16.9 ACTION

* Add the following paragraph after **[O-217ca]** in **§16.9 ACTION**.

> [O-???ca] An application SHOULD check bit 16 (*tagged_action_command*) of the `GVCP Capability `register at address 0x0934 to check if Tagged Action Commands are supported by the device.

* Change **Figure 16-17: ACTION_CMD Message** in **16.9.1 ACTION_CMD**, from

> ```
> |0           |          15|16          |          31|
> |    0x42    |    flag    |   command = ACTION_CMD  |
> |          length         |          req_id         |
> |                     device_key                    |
> |                     group_key                     |
> |                     group_mask                    |
> |             [action_time (high part)]             |
> |              [action_time (low part)]             |
> ```

to

> ```
> |0           |          15|16          |          31|
> |    0x42    |    flag    |   command = ACTION_CMD  |
> |          length         |          req_id         |
> |                     device_key                    |
> |                     group_key                     |
> |                     group_mask                    |
> |             [action_time (high part)]             |
> |              [action_time (low part)]             |
> | [tag_flags]|[tag_offset]|       [tag_length]      |
> |                         ⋮                         |
> |            [tag_data (variable length)]           |
> |                         ⋮                         |
> ```

* Change the description of "flag" field in the table, from

> - bit 0 – Action time available (Scheduled Action Commands)
>   - 0: *action_time* field is not available, GVCP packet is 20 bytes.
>   - 1: *action_time* field (64 bits) is available, GVCP packet is 28 bytes.
> - bit 1 to 3 – Reserved. Set to 0 on transmission, ignore on reception.
> - bit 4 to 7 – Use standard definition from GVCP header

to

> - bit 0 – Action time available (Scheduled Action Commands)
>   - 0: *action_time* field is not available, *tag_flags* field is at offset 20 (if exists).
>   - 1: *action_time* field (64 bits) is available, *tag_flags* field is at offset 28 (if exists).
> - bit 1 – Action tag available (Tagged Action Commands)
>   - 0: *tag_flags*, *tag_offset* and *tag_length* fields are not available.
>   - 1: *tag_flags*, *tag_offset* and *tag_length* fields are available.
> - bit 2 to 3 – Reserved. Set to 0 on transmission, ignore on reception.
> - bit 4 to 7 – Use standard definition from GVCP header

* Add these rows to the end of the table:

> |...|...|...
> |:--|:--|:--
> |tag_flags|8 bits|Reserved. Set to 0 on transmission, ignore on reception.
> |tag_offset|8 bits|The number of 4-byte units from the beginning of the GVCP header (0x42 byte) to *tag_data*. The actual offset is `4 * tag_offset` in bytes. If this field is 0, 0x42 is the first byte of *tag_data* (assuming `tag_length` > 0).
> |tag_length|16 bits|The length of *tag_data*, in bytes.
> |tag_data|*tag_length* bytes|The tag data.

#### 3.1.3 Changes to §27.29 GVCP Capability Register

* In the per-bit-name table, fill the cell below "16" with "TAC"。

* In the description table, change this row:

> |...|...|...
> |:--:|:--|:--
> |16 – 24|reserved|Always 0. Ignore when reading.

to:

> |...|...|...
> |:--:|:--|:--
> |16|tagged_action_command (TAC)|Tagged Action Commands are recognized and consumed. The device does not necessarily have tag related function.
> |17 – 24|reserved|Always 0. Ignore when reading.

### 3.2 Revision to the GenICam SFNC standard

These changes are based on: *GenICam Standard Features Naming Convention (SFNC), Version 2.7.1*

#### 3.2.1 Changes to §2 Features Summary

* Add these rows at the end of **Table 2-12: Action Control Summary**:

> |Name|Level|Interface|Access|Unit|Visibility|Description
> |:--|:--:|:--:|:--:|:--:|:--:|:--
> |ActionTagChunkOffset [ActionSelector]|O|IInteger|R/W|-|G|Provides the number of bytes that will be truncated from the beginning of the action tag to form ChunkActionTag.
> |ActionTagChunkBudget [ActionSelector]|O|IInteger|R/W|-|G|Provides the maximum number of bytes that will be extracted from the action tag to form ChunkActionTag.

* Add these rows at the end of **Table 2-22: Chunk Data Control Summary**:

> |Name|Level|Interface|Access|Unit|Visibility|Description
> |:--|:--:|:--:|:--:|:--:|:--:|:--
> |ChunkActionSelector|O|IInteger|R/W|-|E|Selects which action to retrieve data from.
> |ChunkActionTag [ChunkActionSelector]|O|IRegister|R|-|E|Returns the tag piece extracted from the tag of the action which caused the acquisition of this payload.

#### 3.2.2 Changes to §14.2 Action Control Features

Add these sections to the end of **§14.2 Action Control Features**.

* Add section for **ActionTagChunkOffset**

> ### 14.2.8 ActionTagChunkOffset
> |Name|ActionTagChunkOffset[ActionSelector]
> |:--|:--
> |Category|ActionControl
> |Level|Optional
> |Interface|IInteger
> |Access|Read/Write
> |Unit|-
> |Visibility|Guru
> |Values|≥0
> 
> Provides the number of bytes that will be truncated from the beginning of the action tag to form ChunkActionTag. The user can select only a sub segment of the tag data (especially in one-action-triggers-multi-devices use case). This value indicates the extraction starting point, relative to the beginning of the action tag.

* Add section for **ActionTagChunkBudget**

> ### 14.2.9 ActionTagChunkBudget
> |Name|ActionTagChunkBudget[ActionSelector]
> |:--|:--
> |Category|ActionControl
> |Level|Optional
> |Interface|IInteger
> |Access|Read/Write
> |Unit|-
> |Visibility|Guru
> |Values|≥0
> 
> Provides the maximum number of bytes that will be extracted from the action tag to form ChunkActionTag. This value indicates the maximum length of the ChunkActionTag chunk.
> Setting to 0 disables the chunk.
> The implementation is recommended to support setting this feature to a value up to 20 or more.

Note: This feature can be used to enable/disable the chunk, instead of defining a corresponding enumeration value for ChunkSelector and using ChunkEnable to enable/disable the chunk.

#### 3.2.3 Changes to §24 Chunk Data Control

Add these sections to the end of **§24 Chunk Data Control**:

* Add section for **ChunkActionSelector**

> ### 24.77 ChunkActionSelector
> 
> |Name|ChunkActionSelector
> |:--|:--
> |Category|ChunkDataControl
> |Level|Optional
> |Interface|IInteger
> |Access|Read/Write
> |Unit|-
> |Visibility|Expert
> |Values|≥0
> 
> Selects which action to retrieve data from.

* Add section for **ChunkActionTag**

> ### 24.78 ChunkActionTag
> 
> |Name|ChunkActionTag[ChunkActionSelector]
> |:--|:--
> |Category|ChunkDataControl
> |Level|Optional
> |Interface|IRegister
> |Access|Read
> |Unit|-
> |Visibility|Expert
> |Values|Zero or more bytes
> 
> Returns the tag piece extracted from the tag of the action command which caused the acquisition of this payload.
> 
> For example, if events happen in the following order:
> * action[1] command with tag TagA triggers AcquisitionStart signal
> * action[2] command with tag TagB triggers FrameStart signal and causes image X to be obtained
> * action[2] command with tag TagC triggers FrameStart signal and causes image Y to be obtained
> * action[1] command with tag TagD is an over-trigger and is discarded
> * action[2] command with tag TagE triggers FrameStart signal and causes image Z to be obtained
> 
> In image Z's payload, ChunkActionTag[1] would be from TagA and ChunkActionTag[2] would be from TagE.
> 
> In case the action tag is not longer enough (`ActionTagChunkOffset` + `ActionTagChunkBudget` > `tag length`), the value of this feature is shorter than `ActionTagChunkBudget`.
> If the action command does not have a tag, the value is zero bytes in length.
> If the acquisition is not caused by any of the action (e.g. line triggered), the value is zero bytes in length.

## Q&A

TODO
