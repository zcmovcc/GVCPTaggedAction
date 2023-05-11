# GVCP Tagged Action

**Request For Comments**

* **Date**: 2023-05-11
* **Version**: 0.3.0
* **Author**: WenHua Zheng
* **Email**: z at cmov dot cc
* **Latest**: https://github.com/zcmovcc/GVCPTaggedAction

## 1. Abstract

This is a proposal to add *Tagged Action Commands* to GigE Vision standard and to add tag related features to GenICam Standards.

## 2. Introduction

The GigE Vision protocol is a widely used standard for image capturing over Ethernet networks. While the standard *action* commands are powerful, they may not always be sufficient to meet the needs of certain applications. In particular, there may be situations where additional information needs to be transmitted along with an action command, such as a unique identifier or parameter settings. To address this need, we propose the addition of *Tagged Action Commands* to the GigE Vision standard.

Two possible ways have been used in the past to identify each image and determine which triggering resulted in which image. One way involved using a counting mechanism that was often unreliable  due to packet loss or signal noise. The other way required trigger devices and cameras to support the PTP protocol and a timestamp searching algorithm. With the proposed Tagged Action Commands, users can simplify this task by embedding a unique ID directly in the action packet and retrieving it from a *chunk* of the image payload.

In order to provide maximum flexibility, the format and semantics of the tag are intentionally left undefined. This allows users to include any type of content within the tag. As another example, a classification code can be passed to instruct the image receiver to handle each image differently. The camera's primary responsibility is to transport the raw bytes from the action tag to the image receiver in the form of a *chunk*. While out of scope of this proposal, potentially a portion of the tag may also be used to influence the camera's image capture itself.

Section 3.1 outlines the proposed changes to the GigE Vision specification, defining the format of Tagged Action Commands and three Tagged Action Command support levels. Section 3.2 proposes adding tag and chunk related features to the GenICam SFNC standard, enabling users to utilize tags in the form of chunks. Section 4 gives several examples of how operations are carried out at the byte level.

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
> Provides the number of bytes that will be truncated from the beginning of the action tag to form ChunkActionTag. The user can select a sub segment of the tag data (especially in one-action-triggers-multi-devices use case). This value indicates the extraction starting point, relative to the beginning of the action tag.

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

## 4. Examples

This chapter provides some examples using hypothetical camera models to illustrate how the entire mechanism works.

### 4.1 Example 1

Suppose we have a camera model called _Nano100_ that has one action signal named _Action1_. Some of its features are configured as shown in the table below:

|Feature Name|Value Range (by Manufacturer)|Value (by User)
|:--|:--|:--
|ChunkModeActive|{ False, True }|True
|ActionDeviceKey|-|0
|ActionSelector|{ 1 }|1
|ActionGroupKey[ActionSelector=1]|-|0
|ActionGroupMask[ActionSelector=1]|-|0xFFFFFFFF
|ActionTagChunkOffset[ActionSelector=1]|{ 0, 1, 2, 3, ..., 540 }|0
|ActionTagChunkBudget[ActionSelector=1]|{ 0, 1, 2, 3, ..., 20 }|8
|AcquisitionBurstFrameCount|-|1
|TriggerMode[TriggerSelector=FrameBurstStart]|{ Off, On }|On
|TriggerSource[TriggerSelector=FrameBurstStart]|-|Action1

The camera's XML description file contains the following snippet to describe how the tag chunk is embedded in the image payload:
```xml
  <Port Name="ActionTagPort">
    <ChunkID>41544147</ChunkID>
  </Port>
  <Integer Name="ChunkActionSelector" NameSpace="Standard">
    <ToolTip>Selects which action to retrieve data from.</ToolTip>
    <Description>Selects which action to retrieve data from.</Description>
    <DisplayName>Chunk Action Selector</DisplayName>
    <Value>1</Value>
    <Min>1</Min>
    <Max>1</Max>
  </Integer>
  <IntReg Name="ChunkActionTag_Len" NameSpace="Custom">
    <Address>0</Address>
    <Length>1</Length>
    <AccessMode>RO</AccessMode>
    <pPort>ActionTagPort</pPort>
    <Sign>Unsigned</Sign>
    <Endianess>LittleEndian</Endianess>
  </IntReg>
  <Register Name="ChunkActionTag" NameSpace="Standard">
    <ToolTip>The tag piece extracted from the tag of the action command which caused the acquisition of this payload.</ToolTip>
    <Description>The tag piece extracted from the tag of the action command which caused the acquisition of this payload.</Description>
    <DisplayName>Chunk Action Tag</DisplayName>
    <Address>1</Address>
    <pLength>ChunkActionTag_Len</pLength>
    <AccessMode>RO</AccessMode>
    <pPort>ActionTagPort</pPort>
  </Register>
```
A C++ application uses the GenApi library to extract chunk data from the received payload, using the following code:

```c++
    chunk_adapter_Nano100.AttachBuffer(...);

    auto node_ChunkActionSelector = nodemap_Nano100->GetNode("ChunkActionSelector");
    auto integer_ChunkActionSelector = dynamic_cast<GenApi::IInteger *>(node_ChunkActionSelector);
    integer_ChunkActionSelector->SetValue(1);

    auto node_ChunkActionTag = nodemap_Nano100->GetNode("ChunkActionTag");
    auto register_ChunkActionTag = dynamic_cast<GenApi::IRegister *>(node_ChunkActionTag);
    auto length = register_ChunkActionTag->GetLength();
    if (length < 0)
        throw std::runtime_error("invalid register length");
    std::printf("Nano100 action1 tag chunk[%d]: ", (int)length);

    std::vector<char> buffer;
    buffer.resize(length);
    register_ChunkActionTag->Get(buffer.data(), length);
    for (auto c : buffer)
        std::printf("%02X ", c);
```

After acquisition started, a device on the Ethernet sends action commands to the camera.

#### 4.1.1 Action command 1

The first action command is represented in hexadecimal form as follows:
```
42 41 01 00
00 30 00 01
00 00 00 00
00 00 00 00
FF FF FF FF
00 06 00 20
A0 A1 A2 A3
A4 A5 A6 A7
A8 A9 AA AB
AC AD AE AF
B0 B1 B2 B3
B4 B5 B6 B7
B8 B9 BA BB
BC BD BE BF
```
It includes a 32-byte tag `A0 A1 A2 A3 A4 A5 A6 A7 A8 A9 AA AB AC AD AE AF B0 B1 B2 B3 B4 B5 B6 B7 B8 B9 BA BB BC BD BE BF`.
Since `ActionTagChunkBudget` is set to 8, only the first 8 bytes of the tag are extracted to form the chunk.
The resulting GVSP payload would look like this:
```
00 00 00 00 00 00 00 00 49 4D 41 47 00 00 00 08
08 A0 A1 A2 A3 A4 A5 A6 A7 41 54 41 47 00 00 00 09
```
Note that the `08` byte in the second row corresponds to the `ChunkActionTag_Len` variable.

The output of the application will be:
```
Nano100 action1 tag chunk[8]: A0 A1 A2 A3 A4 A5 A6 A7 
```

#### 4.1.2 Action command 2

Another action command is sent with the following content:
```
42 01 01 00
00 0C 00 02
00 00 00 00
00 00 00 00
FF FF FF FF
```
As a result, the following GVSP payload is generated:
```
00 00 00 00 00 00 00 00 49 4D 41 47 00 00 00 08
00 00 00 00 00 00 00 00 00 41 54 41 47 00 00 00 09
```
Subsequently, the application outputs:
```
Nano100 action1 tag chunk[0]: 
```
It should be noted that this action command contains no tag at all, resulting in a chunk of length 0.

#### 4.1.3 Action command 3

Following the previous two action commands, the acquisition is stopped and the `ActionTagChunkOffset` feature is modified from 0 to 2. Afterwards, the acquisition is restarted and the third action command is sent:

```
42 41 01 00
02 1C 00 03
00 00 00 00
00 00 00 00
FF FF FF FF
00 86 00 05
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
C0 C1 C2 C3 C4 C5 C6 C7 00 00 00 00
```
The resulting GVSP payload is as follows:
```
00 00 00 00 00 00 00 00 49 4D 41 47 00 00 00 08
03 C2 C3 C4 00 00 00 00 00 41 54 41 47 00 00 00 09
```
The output of the application is:
```
Nano100 action1 tag chunk[3]: C2 C3 C4 
```

This action command has a `tag_offset` field with a value of 0x86 and a `tag_length` field with a value of 0x0005. So the tag is `C0 C1 C2 C3 C4`. However, since `ActionTagChunkOffset` is set to 2, the first two bytes of the tag are discarded and the resulting chunk is `C2 C3 C4`.

It's worth noting that the total length of this action command is 548, which is the maximum length defined by the GigE Vision standard for GVCP packets. Cameras with a value of 1 in bit 16 (*tagged_action_command*) of the `GVCP Capability` register must support action commands up to this length.

### 4.2 Example 2

Suppose we have the same Nano100 camera as in _§Example 1_, with its `ActionTagChunkOffset` feature set to 2. And we have two other cameras _Mega2000_ and _AT0_.

* Mega2000 has two action signals, *Action_0* and *Action_1*. Some of its features are configured as shown in the table below:

|Feature Name|Value Range (by manufacturer)|Value (by user)
|:--|:--|:--
|ChunkModeActive|{ False, True }|True
|ActionDeviceKey|-|0
|ActionSelector|{ 0, 1 }|0 and 1
|ActionGroupKey[ActionSelector=0]|-|0
|ActionGroupMask[ActionSelector=0]|-|0xFFFFFFFF
|ActionTagChunkOffset[ActionSelector=0]|{ 0, 1, 2, 3, ..., 65507 }|2
|ActionTagChunkBudget[ActionSelector=0]|{ 0, 1, 2, 3, ..., 65507 }|8
|ActionGroupKey[ActionSelector=1]|-|1
|ActionGroupMask[ActionSelector=1]|-|0xFFFFFFFF
|ActionTagChunkOffset[ActionSelector=1]|{ 0, 1, 2, 3, ..., 65507 }|0
|ActionTagChunkBudget[ActionSelector=1]|{ 0, 1, 2, 3, ..., 65507 }|8
|AcquisitionFrameCount|-|3
|TriggerMode[TriggerSelector=AcquisitionStart]|{ Off, On }|On
|TriggerSource[TriggerSelector=AcquisitionStart]|-|Action_0
|TriggerMode[TriggerSelector=FrameStart]|{ Off, On }|On
|TriggerSource[TriggerSelector=FrameStart]|-|Action_1

* AT0 is a _Level 1_ device, which means it recognizes tagged action commands but treats them just like untagged action commands. It has one action signal called _Action1_. Some of its features are configured as shown in the table below:

|Feature Name|Value Range (by manufacturer)|Value (by user)
|:--|:--|:--
|ActionDeviceKey|-|0
|ActionSelector|{ 1 }|1
|ActionGroupKey[ActionSelector=1]|-|0
|ActionGroupMask[ActionSelector=1]|-|0xFFFFFFFF
|AcquisitionBurstFrameCount|-|1
|TriggerMode[TriggerSelector=FrameBurstStart]|{ Off, On }|On
|TriggerSource[TriggerSelector=FrameBurstStart]|-|Action1

All devices are connected to the same Ethernet network, and all the action commands are broadcasted.

The following table shows example commands and the reactions of each camera:

|Event Order|Action Command|AT0|Nano100|Mega2000
|:--|:--|:--|:--|:--
|1|Action(Group 0):<br>`42 41 01 00`<br>`00 14 00 01`<br>`00 00 00 00`<br>`00 00 00 00`<br>`FF FF FF FF`<br>`00 06 00 04`<br>`A0 A1 A2 A3`|-|-|-|
|2|-|produce image<br>without tag chunk|produce image with<br>action1 tag chunk `A2 A3`|wait for frame trigger (action_1)
|3|Action(Group 1):<br>`42 41 01 00`<br>`00 14 00 02`<br>`00 00 00 00`<br>`00 00 00 01`<br>`FF FF FF FF`<br>`00 06 00 04`<br>`B0 B1 B2 B3`|-|-|-|
|4|-|ignore|ignore|produce one image with 2 tag chunks:<br>action_0 tag chunk `A2 A3`,<br>action_1 tag chunk `B0 B1 B2 B3`|
|5|Action(Group 1):<br>`42 41 01 00`<br>`00 14 00 03`<br>`00 00 00 00`<br>`00 00 00 01`<br>`FF FF FF FF`<br>`00 06 00 04`<br>`C0 C1 C2 C3`|-|-|-|
|6|-|ignore|ignore|produce one image with 2 tag chunks:<br>action_0 tag chunk `A2 A3`,<br>action_1 tag chunk `C0 C1 C2 C3`|
|7|Action(Group 0):<br>`42 41 01 00`<br>`00 14 00 04`<br>`00 00 00 00`<br>`00 00 00 00`<br>`FF FF FF FF`<br>`00 06 00 04`<br>`D0 D1 D2 D3`|-|-|-|
|8|-|produce image<br>without tag chunk|produce image with<br>action1 tag chunk `D2 D3`|ignore, because this is an over-trigger<br>(last acquisition has not<br>reached 3 frames yet)
|9|Action(Group 1):<br>`42 41 01 00`<br>`00 14 00 05`<br>`00 00 00 00`<br>`00 00 00 01`<br>`FF FF FF FF`<br>`00 06 00 04`<br>`E0 E1 E2 E3`|-|-|-|
|10|-|ignore|ignore|produce one image with 2 tag chunks:<br>action_0 tag chunk `A2 A3`(**not `D2 D3`**),<br>action_1 tag chunk `E0 E1 E2 E3`|
|11|Action(Group 1):<br>`42 41 01 00`<br>`00 14 00 06`<br>`00 00 00 00`<br>`00 00 00 01`<br>`FF FF FF FF`<br>`00 06 00 04`<br>`F0 F1 F2 F3`|-|-|-|
|12|-|ignore|ignore|ignore,<br>because AcquisitionFrameCount=3<br>and this is the 4th frame trigger

## Q&A

TODO
