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
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#main-processes">Main Processes</a></li>
    <ol>
      <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#checkpointing">Checkpointing</a></li>
      <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#victim-selection">Victim Selection</a></li>
      <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#gc">GC</a></li>
    </ol>
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
    <li>most of the frequently accessed meta data(super block/checkpoint/NAT/SIT except SSA) are versioned(version0 and version1)</li>
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
     </ul>
    </ul>
    <li>SSA is basicly an array of SSA entry, indexed by Segment No</li>
    <li>SSA entry has following information</li>
    <ul>
      <li>512 summary entries for the 512 blocks of the segment</li>
      <li>journal</li>
      <li>segment type : data or node</li>
      <li>check sum</li>
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
      <li>Flag, is consist of the following components</li>
      <ul>
        <li>cold flag</li>
        <li>fsync flag</li>
        <li>dentry flag</li>
        <li>offset in inode, I Node may (usually) have additionally nodes (Direct Node/Indirect Node), they make up a tree structure, I Node is the root. offset numbering the tree structure from top to down and left to right : the offset of I Node is 0, offset of the first and second Direct Node are 1 and 2, offset of the first and second Indirect Node are 3 and 4+1018, and so on...<br>
        <img src="https://user-images.githubusercontent.com/13962657/181226854-a7358bba-d6f8-42e2-8162-ce4f99f44d1c.png" width="250"></img>
        </li>
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
  <td><img src="https://user-images.githubusercontent.com/13962657/180909796-54b0aeaf-9c94-4944-94d0-1131bc9ae9a5.png" width="100"></img></td>
  <td>
    <ol>
      <li>a direct node has 1018 block address entries , it will cover 1018*4K bytes of data</li>
    </ol>
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180909757-6d8e60ac-e0ee-4a9c-86f7-c823f03aba6c.png" width="100"></img></td>
  <td>
    <ol>
        <li>an indirect node has 1018 nid entries, it will cover 1018 direct nodes or 1018 indirect nodes. in the former an indirect node will cover 1018*1018*4K bytes of data, in the latter, an indirect node will cover 1018*1018*1018*4K bytes of data
    </ol>
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
  <td><img src="https://user-images.githubusercontent.com/13962657/181260293-6a143196-e9f8-44e9-bc43-7ef3ecddbe67.png" width="350"></img></td>
  <td>
    <ol>
    <li>locks</li>
    <ul>
      <li>nid_list_lock, a spin lock, is used to protect FreeNID Cache / FreeNIDBitmaps</li>
      <li>nat_tree_lock, an rw lock, is used to protect NatE Cache</li>
      <li>build_lock, a mutex</li>      
    </ul>
    <li>data flows</li>
    <ul>
      <li>green arrow is the data flow of FreeNID Cache building</li>
      <li>red arrow is the data flow of checkpoiting</li>
      <li>light blue arrow is the data flow of NatE Cache loading</li>
    </ul>
    <li>FreeNIDBitmaps</li>
    <ul>
      <li>an array of bitmap, indexed by NAT block No</li>
      <li>is guarded by NATBlockBitmap. to update bitmap of an NAT block, it must has its bit set in NATBlockBitmap</li>
      <li>updated in process of NAT scanning (the light green arrow)</li>
      <li>updated from Full/EmptyBitmap (the blue arrow)</li>
      <li>updated at checkpoint time</li>
      <li>updated at pre-allocating time (clear)</li>
      <li>updated in fail function call (set)</li>
    </ul>
    <li>Full/EmptyBitmap</li>
    <ul>
      <li>these 2 bitmaps are enabled when CP_NAT_BITS_FLAG is set</li>
      <li>if an NAT block is empty, the bit in EmptyBitmap will be set</li>
      <li>if an NAT block is full, the bit in FullBitmap will set set</li>
      <li>updated from FreeNIDBitmaps(the purple arrow) the first time CP_NAT_BITS_FLAG is set</li>
      <li>updated at checkpoint time</li>
    </ul>
    <li>NATBlockBitmap</li>
    <ul>
      <li>used as a guard for updating FreeNIDBitmaps</li>
      <li>used as NAT scanning hint : NAT block set in this bitmap will be skipped</li>
      <li>updated in process of scanning NAT (orange arrow)</li>
      <li>updated from Full/EmptyBitmap (orange arrow)</li>      
    </ul>
    <li>FreeNID Cache</li>
    <ul>
      <li>updated at mount time : f2fs will scan the on-disk NAT and NAT Journal</li>
      <li>updated at run time : when there is not enough free nid, will scan the FreeNIDBitmaps, NAT Jounal and NAT</li>
      <li>updated at check point time : if an NatE Cache becomes free</li>
      <li>updated when f2fs decide that there are too many FreeNID Cache, some amount of entries will be deleted. but the bits of which in FreeNIDBitmaps will keep set, in case of there is not enough FreeNID Cache , the deleted entries can be brought back quickly by scanning FreeNIDBitmaps</li>
    </ul>
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/181168996-3e6181e4-5df9-4b5c-a8cf-bd884f84737f.png" width="350"></img></td>
  <td>
    <ol>
      <li>FreeNID Cache entries are organized in a balanced tree with node id as its key</li>
      <li>at the same time, these entries are linked into a list, entry will be allocated from the head of the list. the allocation is divided into 2 stages</li>
      <ol>
        <li>the first stage is pre-allocation stage : entry is deleted from the list (1)</li>
        <li>the second stage is succeeding/failing stage</li>
        <ul>
          <li>if f2fs decide that the pre-allocation is succeeded, it will call the done function, node manager will delete the entry from the tree (3)</li>
          <li>if f2fs decide that the pre-allocation is failed, it will call the fail function, node manager will delete the entry from the tree(3) or append it back to the list (2)</li>
        </ul>
      </ol>
    </ol>
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
      <li>locks</li>
      <ul>
        <li>segmap_lock, a spin lock, is used to protect FreeSegBitmap / FreeSecBitmap</li>
        <li>sentry_lock, a rw lock, is used to protect SitE Cache</li>
        <li>journal_rwsem, a rw lock, is used to protect NAT/SIT journal in CurSegs[n]</li>
        <li>seglist_lock, a mutex, is used to protect DirtySegBitmaps / DirtySecBitmap / VictimSecBitmap</li>
        <li>curseg_mutex, a mutex, is used to protect CurSegs[n]</li>        
      </ul>
      <li>data flows</li>
      <ul>
        <li>red arrow is the data flow of checkpointing</li>
        <li>green arrow is the data flow of SitE Cache loadinig</li>
        <li>light blue arrow is the data flow of segment to section bitmap rollup / dirty SitE bitmap update</li>
        <li>blue arrow is the data flow of segment switching</li>
        <li>orange arrow is the data flow of dirty segment update</li>
      </ul>
      <li>DirtySegBitmaps is consist of 8 bitmaps</li>
      <ul>
        <li>the DIRTY bitmap, is the union of the following 6 sub dirty bitmaps, any 2 sub dirty bitmaps have no intersection</li>
        <ul>
          <li>hot data dirty bitmap</li>
          <li>warm data dirty bitmap</li>
          <li>cold data dirty bitmap</li>
          <li>hot node dirty bitmap</li>
          <li>warm node dirty bitmap</li>
          <li>cold node dirty bitmap</li>
        </ul>
        <li>the PRE bitmap, the prefreed segment has its bit set in this bitmap. Dirty segment may become free in the process of block address release or GC. These kind of segment is recorded in this PRE bitmap. At checkpoint time, the prefreed segment(s) will move to free segment(s) (PRE DirtySegBitmap->FreeSegBitmap)</li>
      </ul>
      <li>CurSegs, an array of current segments, f2fs allocate node or data from current segment, there are 8 types of current segment</li>
      <ul>
        <li>hot data</li>
        <li>warm data</li>
        <li>cold data</li>
        <li>hot node</li>
        <li>warm node</li>
        <li>cold node</li>
        <li>pinned cold data</li>
        <li>atgc</li>
      </ul>
    </ol>
  </td>
</tr>
</table>

## main processes
### checkpointing
### victim selection
### gc



