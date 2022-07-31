# Summary
this repo is notes of Linux f2fs file system in my preparation of porting f2fs to Windows.
# Table of Contents

<ol>
  <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#f2fs">F2FS</a></li>
  <ol>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#disk-layout">disk layout</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#checkpoint">checkpoint</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#natsitssa">NAT/SIT/SSA</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#node">node</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#node-config">node config, file size, etc.</a></li>
  </ol>
  <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#linux-implementation">Linux Implementation</a></li>
  <ol>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#locks">Locks</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#node-manager">Node Manager</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#segment-manager">Segment Manager</a></li>
    <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#main-processes">Main Processes</a></li>
    <ol>
      <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#checkpointing">checking point</a></li>
      <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#victim-selection">victim selection</a></li>
      <li><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#gc">gc</a></li>
    </ol>
  </ol>
</ol>

# F2FS
## disk layout
<table>
<tr><td width="25%">figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/181773789-6b09d493-79ac-4777-81f6-9838eab1e44e.png" height="350"></img></td>
  <td>
  <ol>
    <li>meta data</li>
    <ul>
      <li>Super Block</li>
      <li>Checkpoint</li>
      <li>SIT, segment information table</li>
      <li>NAT, node address table</li>
      <li>SSA, segment summary area</li>
    </ul>
    <li>frequently accessed meta datas(Super Block / Checkpoint / SIT / NAT, except SSA) are versioned(there are 2 versions)</li>
    <ul>
      <li>the purpose of meta data versioning is to balance the wirte of meta area</li>
      <li>version switching happened at checking point time</li>
    </ul>
    <li>Super Block, has following important information</li>
    <ul>
      <li>magic number, is used to recognize f2fs volume</li>
      <li>major and minor version of f2fs</li>
      <li>sector size, block size</li>
      <li>sectos per block, blocks per segment (512), segments per section (usually 1, settable by mkfs.f2fs), sections per zone (usually 1)</li>
    </ul>
    <li>SIT is basicly an array of SIT entries , indexed by Segment No</li>
    <li>SIT entry has following information</li>
    <ul>
     <li>allocated block count of the segment</li>
     <li>bitmap of allocated blocks</li>
     <li>segment temperature</li>
     <li>the average access time of the segment</li>
     <li> refer to <a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#natsitssa">nat/sit/ssa</a> for detail</li>
    </ul>
    <li>NAT is basicly an array of NAT entries , indexed by Node ID</li>
    <li>NAT entry has following information</li>
    <ul>
     <li>version</li>
     <li>I Node identifier</li>
     <li>block address</li>
      <li> refer to <a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#natsitssa">nat/sit/ssa</a> for detail</li>
    </ul>
    <li>SSA is basicly an array of SSA entry, indexed by Segment No. the size of SSA entry is 4K</li>
    <li>SSA entry has following information</li>
    <ul>
      <li>summary entries</li>
      <li>journal</li>
      <li>segment type</li>
      <li>check sum</li>
      <li>refer to <a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#natsitssa">nat/sit/ssa</a> for detail
    </ul>
  <ol>
  </td>
</tr>
</table>

## checkpoint
<table>
<tr><td width="35%">figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180909367-adb528c9-49a5-46bd-b245-f8c2d65636e9.png" height="350"></img></td>
  <td>
    <ol>
      <li>Header and Footer, they are indentical if this is a valid checkpoint, have following information</li>
      <ul>
        <li>size (in unit of block) of checkpoint : from Header to Footer</li>
        <li>current segments : segment No, next block offset, allocation type etc.</li>
        <li>NAT version bitmap</li>
        <li>SIT version bitmap</li>
      </ul>
      <li>Payload, if Header can not accommodate the SIT version bitmap, it will be stored in Payload, or else this area is empty</li>
      <li>Orphan inode, if any, or else this area is empty</li>
      <li>DataSummary, summaries of the current data segments, has 2 formats</li>
        <ul>
          <li>compact format : if size of NAT journal + SIT journal + SSA summary entries is less than 3 blocks</li>
            <ol>
              <li>NAT journal</li>
              <li>SIT journal</li>
              <li>SSA summary entries for hot data</li>
              <li>SSA summary entries for warm data</li>
              <li>SSA summary entries for cold data</li>
            </ol>
          <li>normal format</li>
          <ol>
            <li>SSA entry for hot data</li>
            <li>SSA entry for warm data</li>
            <li>SSA entry for cold data</li>
          </ol>
        </ul>
      <li>NodeSummary : summaries of the current node segments if checking point reason is CP_UMOUNT or CP_FASTBOOT, or else this area is empty</li>
      <ol>
        <li>SSA entry for hot node</li>
        <li>SSA entry for warm node</li>
        <li>SSA entry for cold node</li>
      </ol>
      <li>NATBits, if enabled, or else this area is empty</li>
    </ol>
  </td>
</tr>
</table>

## NAT/SIT/SSA
<table>
<tr><td>figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180914285-503a452c-2aed-44b9-baa5-67b1f5b7f319.png" width="240"></img></td>
  <td>
    <ol>
      <li>NAT is organized in 4K sized blocks, an NAT block accommodates 455 NAT entries</li>
      <li>Version, every time the BlkAddr is changed from non-NULL_ADDR to NULL_ADDR , Version will increase by 1</li>
      <li>INO</li>
      <li>BlkAddr</li>
      <ul>
        <li>NULL_ADDR, the NAT entry is free for allocating</li>
        <li>NEW_ADDR, the NAT entry is allocated but node is not allocated</li>        
        <li>COMPRESS_ADDR</li>
        <li>Used (none of the above), the NAT entry is allocated and node is allocated, BlkAddr is the address of the node</li>
      </ul>
    </ol>
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180914330-e21e72c3-1f55-4f6e-b4c1-70768703738d.png" width="240"></img></td>
  <td>
    <ol>
      <li>SIT is organized in 4K sized blocks, a SIT block accommodates 55 SIT entries</li>
      <li>Blocks, is consist of the following components</li>
      <ul>
        <li>bits 0-9, is the allocated block count in the segment</li>
        <li>bits 10-15, is the segment temperature : hot/warm/cold</li>
      </ul>
      <li>BlockBitmap, every allocated block in this segment has its bit set in this bitmap</li>
      <li>Mtime, average access time of the segment</li>
      <ul>
        <li>when a block in this segment is allocated or freed, the access time is calculated and averaged with Mtime</li>
        <li>is used in victim segment selection</li>
      </ul>
    </ol>
  </td>
</tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/180914357-1ead86e6-ce22-46c1-805f-c9a6fc66b997.png" width="240"></img></td>
  <td>
    <ol>
      <li>SSA summary entry is the summary of a block in a segment, there are 512 such entries</li>
      <ul>
        <li>NID</li>
        <li>Version</li>
        <ul>
          <li>copy of NAT entry version of Direct Node if this is a data block</li>
          <li>0 if this is a node block</li>
        </ul>
        <li>Offset</li>
        <ul>
          <li>offset in Direct Node if this is a data block</li>
          <li>0 if this is a node block</li>
        </ul>
      </ul>
      <li>Footer</li>
      <ul>
        <li>Type: data or node. f2fs does not mix data block and node block in the same segment </li>
      </ul>
    </ol>
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
        <li>bit 0 : cold flag</li>
        <li>bit 1 : fsync flag</li>
        <li>bit 2 : dentry flag</li>
        <li>bit 3-31 : offset, refer to <a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#node-config">node config</a> for detail</li>
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
        <li>the other entries (InlineDataAddrs) are used to store addresses of inlined data or inlined data
      </ul>
      <li>NID, if file size can not fit in InlineDataAddrs completely, f2fs will allocate additional nodes, refer to <a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#node-config">node config</a></li>
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

## node config
<table>
<tr><td width="40%">figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/181226854-a7358bba-d6f8-42e2-8162-ce4f99f44d1c.png" width="400"></img></td>
  <td>
    <ol>
      <li>f2fs use 5 configurations to accommodate different file sizes, this makes up a tree structure</li>
      <ul>
      	<li>blue circle is I Node (file)</li>
      	<li>light green circle is Indirect Node (I)</li>
      	<li>green circle is Direct Node (D)</li>
      </ul>
      <li>file size and configuration(Isz is the size that can be inlined, N==1018)</li>
      <table>
        <tr><td>file size</td><td>node config (cumulative)</td><td>NID Index</td></tr>
        <tr><td><=Isz</td><td>I Node</td><td>-</td></tr>
        <tr><td><=Isz+N*4K</td><td>D</td><td>0</td></tr>
        <tr><td><=Isz+N*4K*2</td><td>D</td><td>1</td></tr>
        <tr><td><=Isz+N*N*4K</td><td>I+D</td><td>2</td></tr>
        <tr><td><=Isz+N*N*4K*2</td><td>I+D</td><td>3</td></tr>
        <tr><td><=Isz+N*N*N*4K</td><td>I+I+D</td><td>4</td></tr>
      </table>
      <li>for example, if file size is between Isz and Isz+1018*4K, f2fs will allocate a Direct Node. portion of the file size that is equal to Isz is stored in I Node, portion that is greater than Isz is store in Direct Node.
      <li>there are 2 kind of offsets in a file : node offset and data offset</li>
      <ul>
      	<li>node offset, is the numbering of the tree structure from top to down and left to right : the offset of I Node is 0, offset of the first and second Direct Node are 1 and 2, offset of the first and second Indirect Node are 3 and 4+1018, and so on...</li>
      	<li>data offset</li>
      </ul>
    </ol>
  </td>
</tr>
</table>

# Linux Implementation
## Locks
<table>
  <tr><td>name</td><td>type</td><td>description</td></tr>
  <tr><td>cp_lock</td><td>spin</td><td>description</td></tr>
  <tr><td>cp_global_sem</td><td>RW</td><td>description</td></tr>
  <tr><td>cp_rwsem</td><td>RW</td><td>write, when checking point<br>read, when fs operation</td></tr>
  <tr><td>locks on Node Manager</td><td>-</td><td><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#node-manager">Node Manager</a></td></tr>
  <tr><td>locks on Segment Manager</td><td>-</td><td><a href="https://github.com/oldoldman/f2fs-design/blob/main/README.md#segment-manager">Segment Manager</a></td></tr>
</table>

## Node Manager
<table>
<tr><td width="40%">figure</td><td>description</td></tr>
<tr valign="top">
  <td><img src="https://user-images.githubusercontent.com/13962657/181406039-246ff912-4441-4db4-98cb-5f1acb819b0f.png" width="350"></img></td>
  <td>
    <ol>
    <li>locks</li>
    <table>
      <tr><td>name</td><td>type</td><td>protecting</td></tr>
      <tr><td>nid_list_lock</td><td>spin</td><td>FreeNID Cache<br> FreeNIDBitmaps</td></tr>
      <tr><td>nat_list_lock</td><td>spin</td><td>NatE Cache entry LRU</td></tr>
      <tr><td>nat_tree_lock</td><td>RW</td><td>NatE Cache</td></tr>
      <tr><td>build_lock</td><td>mutex</td><td>NATBlockBitmap</td></tr>
    </table>
    <li>data flows</li>
    <ul>
      <li>green arrow is the data flow of FreeNID Cache building</li>
      <li>red arrow is the data flow of checkpoiting</li>
      <li>light blue arrow is the data flow of NatE Cache loading</li>
    </ul>
    <li>FreeNIDBitmaps, an array of bitmap, indexed by NAT block No, each bitmap is indexed by offset of nid in NAT block</li>
    <ul>
      <li>is guarded by NATBlockBitmap : to update bit in bitmap of an NAT block, bit of which in NATBlockBitmap must be set</li>
      <li>updated in NAT scanning (the light green arrow)</li>
      <li>updated from NATBits (the blue arrow)</li>
      <li>updated at checking point time</li>
      <li>updated at pre-allocation time (clear)</li>
      <li>updated in fail function call (set)</li>
      <li>used in run time FreeNID Cache building</li>
    </ul>
    <li>NATBits, is consist of 2 bitmaps : FullBitmap and EmptyBitmap, both indexed by NAT block No</li>
    <ul>
      <li>is enabled when CP_NAT_BITS_FLAG is set</li>
      <li>if an NAT block is empty (all NAT entries are free, NULL_ADDR), the bit in EmptyBitmap will be set</li>
      <li>if an NAT block is full (all NAT entries are used, non-NULL_ADDR), the bit in FullBitmap will be set</li>
      <li>a dirty NAT block will has its bit set neither in EmptyBitmap nor in FullBitmap</li>
      <li>updated from FreeNIDBitmaps (the purple arrow) the first time NATBits is enabled</li>
      <li>updated at checking point time</li>
    </ul>
    <li>NATBlockBitmap, indexed by NAT block No</li>
    <ul>
      <li>used as a guard for updating FreeNIDBitmaps</li>
      <li>used as NAT scanning hint : NAT block set in this bitmap will be skipped</li>
      <li>updated in process of scanning NAT (orange arrow)</li>
      <li>updated from NATBits (orange arrow)</li>      
    </ul>
    <li>FreeNID Cache</li>
    <ul>
      <li>updated at mount time : f2fs will scan the NAT and NAT Journal</li>
      <li>updated at run time : when there is not enough free nid, f2fs will scan the FreeNIDBitmaps, NAT Jounal and NAT</li>
      <li>updated at checking point time : if an NatE Cache entry becomes free</li>
      <li>updated when f2fs decide that there are too many FreeNID Cache, portion of the entries will be deleted, but the bits of which in FreeNIDBitmaps will keep set, in case of there is not enough FreeNID Cache, the deleted entries can be brought back quickly by scanning FreeNIDBitmaps</li>
    </ul>
    <li>NatE Cache</li>
    <ul>
      <li>an NatE Cache entry is added on-demand</li>
      <ul>
        <li>added in the code path of f2fs_get_node_info</li>
        <li>added in the code path of set_node_addr</li>
      </ul>
      <li>an entry becomes dirty when its address changed</li>
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
      <table>
        <tr><td>name</td><td>type</td><td>protect</td></tr>
        <tr><td>segmap_lock</td><td>spin</td><td>FreeSegBitmap<br> FreeSecBitmap</td></tr>
        <tr><td>sentry_lock</td><td>RW</td><td>SitE Cache</td></tr>
        <tr><td>journal_rwsem</td><td>RW</td><td>NAT/SIT journal in CurSegs[n]</td></tr>
        <tr><td>seglist_lock</td><td>mutex</td><td>DirtySegBitmaps<br> DirtySecBitmap<br> VictimSecBitmap</td></tr>
        <tr><td>curseg_mutex</td><td>mutex</td><td>CurSegs[n]</td></tr>
      </table>
      <li>data flows</li>
      <ul>
        <li>red arrow is the data flow of checkpointing</li>
        <li>green arrow is the data flow of SitE Cache loadinig</li>
        <li>light blue arrow is the data flow of segment to section bitmap rollup / dirty SitE bitmap update</li>
        <li>blue arrow is the data flow of segment switching</li>
        <li>orange arrow is the data flow of dirty segment update</li>
      </ul>
      <li>DirtySegBitmaps is consist of 8 bitmaps, all are indexed by segment No</li>
      <ul>
        <li>the DIRTY bitmap, is the union of the following 6 sub-DIRTY bitmaps, any 2 sub-DIRTY bitmaps have no intersection</li>
        <ul>
          <li>hot data dirty bitmap</li>
          <li>warm data dirty bitmap</li>
          <li>cold data dirty bitmap</li>
          <li>hot node dirty bitmap</li>
          <li>warm node dirty bitmap</li>
          <li>cold node dirty bitmap</li>
        </ul>
        <li>the PRE bitmap, the prefreed segment has its bit set in this bitmap. dirty segment may become free in the process of block address release or GC. these kind of segment is recorded in this PRE bitmap. at checking point time, the prefreed segment(s) is moved to free segment(s) (PRE DirtySegBitmap->FreeSegBitmap)</li>
      </ul>
      <li>CurSegs, an array of current segments, f2fs allocate node or data from current segments, there are 8 types of current segment</li>
      <ul>
        <li>hot data</li>
        <li>warm data</li>
        <li>cold data</li>
        <li>hot node, from which Direct Node is allocated</li>
        <li>warm node</li>
        <li>cold node, from which Indriect Node is allocated</li>
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



