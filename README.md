# Summary
this repo is notes of Linux f2fs file system in my preparation of porting f2fs to Windows.
# Table of Contents

<ol>
  <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#f2fs">F2FS</a></li>
  <ol>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#disk layout overview">disk layout</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#checkpoint">checkpoint</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#node">node</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#natsitssa">nat/sit/ssa</a></li>
  </ol>
  <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#linux-implementation">Linux Implementation</a></li>
  <ol>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#node-manager">Node Manager</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#segment-manager">Segment Manager</a></li>
  </ol>
</ol>

# F2FS
## disk layout overview
<table>
<tr><td width="25%">figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180909202-51e07d8a-cc8c-46e6-ba44-86ab55996301.png" height="350"></img></td>
  <td>
  <ol>
    <li>most meta data(super block/checkpoint/NAT/SIT except SSA) are versioned(version0 and version1)</li>
    <ul>
      <li>the purpose of meta data versioning is to balance the wirte of meta area</li>
      <li>version switching happened at checkpointing</li>
    </ul>
    <li>SIT is basicly an array of SIT entries , indexed by Segment No</li>
    <li>SIT entry has following information</li>
    <ul>
     <li>allocated block count of the segment</li>
     <li>bitmap of allocated blocks</li>
     <li>segment type : hot/warm/cold</li>
     <li>the average access time of the segment , which is used in victim segment selection</li>
     <li>when a block is allocated or freed , the access time is calculated and averaged with the segment average access time</li>
    </ul>
    <li>NAT is basicly an array of NAT entries , indexed by Node ID</li>
    <li>NAT entry has following information</li>
    <ul>
     <li>NAT entry version , every time the node block address is changed from non-NULL_ADDR to NULL_ADDR , the version will increase by 1</li>
     <li>inode id</li>
     <li>node block address</li>
     <ul>
       <li>Used , the NAT entry is allocated and node has allocated</li>
       <li>NULL_ADDR, the NAT entry is free for allocating</li>
       <li>NEW_ADDR, the NAT entry is allocated but node is not allocated</li>
       <li>COMPRESS_ADDR</li>
     <ul>
    </ul>
  <ol>
  </td>
</tr>
</table>

## checkpoint
<table>
<tr><td>figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180909367-adb528c9-49a5-46bd-b245-f8c2d65636e9.png" height="350"></img></td>
  <td>
  
  </td>
</tr>
</table>

## node
<table>
<tr><td width="35%">figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180909586-427beb46-c013-4873-8c2e-557ee2d3f853.png" width="220"></img></td>
  <td>
    <ol>
      <li>NID is the node identifier</li>
      <li>INO is the I Node identifier the node belong to. If NID==INO this is an I Node</li>
      <li>Flag</li>
      <ul>
        <li>cold flag</li>
        <li>fsync flag</li>
        <li>dentry flag</li>
        <li>offset in inode</li>
      </ul>
      <li>CkpVer</li>
      <li>NextBlkAddr</li>
    </ol>
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180909701-02553dbb-af67-47e2-a951-3a08781db68e.png" width="220"></img></td>
  <td>
    <ol>
      <li>InlineDataAddrs, this is an array of 923 entries , each entry is 4 bytes</li>
      <ul>
        <li>optionally, this array can be used to store extra attributes inline (ExtraAttr)</li>
        <li>optionally, this array can also be used to store extension attributes inline (InlineXAttr)
        <li>the other entries (InlineDataAddrs) are used to store address of inlined file data
      </ul>
      <li>NID, if file size can not fit in InlineDataAddrs completely, f2fs will allocate additional nodes as follow (ISIZE is the size that can be inlined)</li>
      <table>
        <tr><td>file size</td><td>node config (cumulated)</td><td>NID Index</td></tr>
        <tr><td><=ISIZE</td><td>I Node</td></tr>
        <tr><td><=ISIZE+1018*4K</td><td>Direct Node</td><td>0</td></tr>
        <tr><td><=ISIZE+1018*4K*2</td><td>Direct Node</td><td>1</td></tr>
        <tr><td><=ISIZE+1018*1018*4K</td><td>Indirect Node+Direct Node</td><td>2</td></tr>
        <tr><td><=ISIZE+1018*1018*4K*2</td><td>Indirect Node+Direct Node</td><td>3</td></tr>
        <tr><td><=ISIZE+1018*1018*1018*4K</td><td>Indirect Node+Indirect Node+Direct Node</td><td>4</td></tr>
      </table>
    </ol>
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180909757-6d8e60ac-e0ee-4a9c-86f7-c823f03aba6c.png" width="100"></img></td>
  <td>
  
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180909796-54b0aeaf-9c94-4944-94d0-1131bc9ae9a5.png" width="100"></img></td>
  <td>
  
  </td>
</tr>
</table>

## nat/sit/ssa
<table>
<tr><td>figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180914285-503a452c-2aed-44b9-baa5-67b1f5b7f319.png" width="220"></img></td>
  <td>
  
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180914330-e21e72c3-1f55-4f6e-b4c1-70768703738d.png" width="220"></img></td>
  <td>
  
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180914357-1ead86e6-ce22-46c1-805f-c9a6fc66b997.png" width="220"></img></td>
  <td>
  
  </td>
</tr>
</table>

# Linux Implementation
## Node Manager
<table>
<tr><td width="40%">figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180929385-82321194-585d-451d-8129-8c8395aee4f3.png" width="350"></img></td>
  <td>
    <ol>
    <li>locks</li>
    <ul>
      <li>build_lock, a mutex</li>
      <li>nat_tree_lock, a r/w lock, is used to protect NatE Cache</li>
      <li>nid_list_lock, a spin lock, is used to protect FreeNode Cache/FreeNodeBitmap</li>
    </ul>
    <li>data flows</li>
    <ul>
      <li>green arrow is the data flow of FreeNode Cache building</li>
      <li>red arrow is the data flow of checkpoiting</li>
      <li>blue arrow is the data flow of NatE Cache loading</li>
    </ul>
    <li>FreeNode Cache building, this process happened at</li>
    <ul>
      <li>mount time, will scan the on-disk NAT and NAT Journal</li>
      <li>run time, when there is not enough free node, will scan the FreeNodeBitmap and NAT Jounal</li>
    </ul>
  </td>
</tr>
</table>

## Segment Manager
<table>
<tr><td width="40%">figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180911020-f763e341-04a5-455c-8345-886f58c37254.png" width="380"></img></td>
  <td>
    <ol>
      <li>CurSegs, an arry of curernt segments, f2fs allocate from curernt segment, there are 8 types of current segment</li>
      <ul>
        <li>hot data</li>
        <li>warm data</li>
        <li>cold data</li>
        <li>hot node</li>
        <li>warm node</li>
        <li>cold node</li>
      </ul>
      <li>current segments can be selected from free segments or dirty segments</li>
    </ol>
  </td>
</tr>
</table>



