# F2FS
## disk layout
|figure|description|
|------|-----------|
|<img src="https://user-images.githubusercontent.com/13962657/180909202-51e07d8a-cc8c-46e6-ba44-86ab55996301.png" height="350"></img>|1. All meta datas(super block/checkpoint/NAT/SIT/SSA) are versioned(version0 and version1) 2. version switch happened at checkpointing|
## check point
|figure|description|
|------|-----------|
|<img src="https://user-images.githubusercontent.com/13962657/180909367-adb528c9-49a5-46bd-b245-f8c2d65636e9.png" height="350"></img>| |
## node
|figure|description|
|------|-----------|
|<img src="https://user-images.githubusercontent.com/13962657/180909586-427beb46-c013-4873-8c2e-557ee2d3f853.png" width="220"></img>|NodeBlk|
|<img src="https://user-images.githubusercontent.com/13962657/180909701-02553dbb-af67-47e2-a951-3a08781db68e.png" width="220"></img>|Inode|
|<img src="https://user-images.githubusercontent.com/13962657/180909757-6d8e60ac-e0ee-4a9c-86f7-c823f03aba6c.png" width="100"></img>|IndirectNode|
|<img src="https://user-images.githubusercontent.com/13962657/180909796-54b0aeaf-9c94-4944-94d0-1131bc9ae9a5.png" width="100"></img>|DirectNode|
## nat/sit/ssa
|figure|description|
|------|-----------|
|<img src="https://user-images.githubusercontent.com/13962657/180914285-503a452c-2aed-44b9-baa5-67b1f5b7f319.png" width="220"></img>|NAT|
|<img src="https://user-images.githubusercontent.com/13962657/180914330-e21e72c3-1f55-4f6e-b4c1-70768703738d.png" width="220"></img>|SIT|
|<img src="https://user-images.githubusercontent.com/13962657/180914357-1ead86e6-ce22-46c1-805f-c9a6fc66b997.png" width="220"></img>|SSA|
# Linux Implementation
## Node Manager
|figure|description|
|------|-----------|
|<img src="https://user-images.githubusercontent.com/13962657/180910969-4ca85a4b-c413-4cbb-a189-1cbeb799a2fc.png" width="300"></img>|Node Manager|
## Segment Manager
|figure|description|
|------|-----------|
|<img src="https://user-images.githubusercontent.com/13962657/180911020-f763e341-04a5-455c-8345-886f58c37254.png" width="330"></img>|Segment Manager|


