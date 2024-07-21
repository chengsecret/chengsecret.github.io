---
title: "CreateIndex执行流程"
subtitle: ""
date: 2024-07-21T22:25:48+08:00
lastmod: 2024-07-21T22:25:48+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["milvus"]
tags: ["go", "向量数据库"]
featuredImage: ""
---

![架构图](/milvus/createIndex/架构.png)

## 数据流程

创建索引的整体数据流向图：

![数据流](/milvus/数据流向.jpg)

## etcd元数据操作

### 1.客户端sdk发出CreateIndex API请求

```PYTHON
import numpy as np
from pymilvus import (
    connections,
    FieldSchema, CollectionSchema, DataType,
    Collection,
)

num_entities, dim = 2000, 8


print("start connecting to Milvus")
connections.connect("default", host="192.168.230.71", port="19530")


fields = [
    FieldSchema(name="pk", dtype=DataType.VARCHAR, is_primary=True, auto_id=False, max_length=100),
    FieldSchema(name="random", dtype=DataType.DOUBLE),
    FieldSchema(name="embeddings", dtype=DataType.FLOAT_VECTOR, dim=dim)
]

schema = CollectionSchema(fields, "hello_milvus is the simplest demo to introduce the APIs")

print("Create collection `hello_milvus`")
hello_milvus = Collection("hello_milvus", schema, consistency_level="Strong")


print("Start inserting entities")
rng = np.random.default_rng(seed=19530)
entities = [
    # provide the pk field because `auto_id` is set to False
    [str(i) for i in range(num_entities)],
    rng.random(num_entities).tolist(),  # field random, only supports list
    rng.random((num_entities, dim)),    # field embeddings, supports numpy.ndarray and list
]

insert_result = hello_milvus.insert(entities)

hello_milvus.flush()

print("Start Creating index IVF_FLAT")
index = {
    "index_type": "IVF_FLAT",
    "metric_type": "L2",
    "params": {"nlist": 8},
}

hello_milvus.create_index("embeddings", index,index_name="idx_embeddings")
```

### 2.服务端接受API请求，将request封装为createIndexTask，并压入ddQueue队列。

代码路径:internal\proxy\impl.go

```go
// CreateIndex create index for collection.
func (node *Proxy) CreateIndex(ctx context.Context, request *milvuspb.CreateIndexRequest) (*commonpb.Status, error) {
	......
    
	// request封装为task
	cit := &createIndexTask{
		ctx:                ctx,
		Condition:          NewTaskCondition(ctx),
		req:                request,
		rootCoord:          node.rootCoord,
		datacoord:          node.dataCoord,
		replicateMsgStream: node.replicateMsgStream,
	}

	......
    // 将task压入ddQueue队列

	if err := node.sched.ddQueue.Enqueue(cit); err != nil {
		......
	}

	......
    // 等待cct执行完
	if err := cit.WaitToFinish(); err != nil {
		......
	}

	......
}
```

### 3.执行createIndexTask的3个方法PreExecute、Execute、PostExecute

- PreExecute()一般为参数校验等工作。
- Execute()一般为真正执行逻辑。
- 代码路径:internal\proxy\task_index.go

```go
func (cit *createIndexTask) Execute(ctx context.Context) error {
	......
	req := &indexpb.CreateIndexRequest{
		CollectionID:    cit.collectionID,
		FieldID:         cit.fieldSchema.GetFieldID(),
		IndexName:       cit.req.GetIndexName(),
		TypeParams:      cit.newTypeParams,
		IndexParams:     cit.newIndexParams,
		IsAutoIndex:     cit.isAutoIndex,
		UserIndexParams: cit.newExtraParams,
		Timestamp:       cit.BeginTs(),
	}
	cit.result, err = cit.datacoord.CreateIndex(ctx, req)
	......
	SendReplicateMessagePack(ctx, cit.replicateMsgStream, cit.req)
	return nil
}
```

从代码可以看出调用了datacoord的CreateIndex接口。

### 4.进入datacoord的CreateIndex接口

代码路径:internal\datacoord\index_service.go

```go
// CreateIndex create an index on collection.
// Index building is asynchronous, so when an index building request comes, an IndexID is assigned to the task and
// will get all flushed segments from DataCoord and record tasks with these segments. The background process
// indexBuilder will find this task and assign it to IndexNode for execution.
func (s *Server) CreateIndex(ctx context.Context, req *indexpb.CreateIndexRequest) (*commonpb.Status, error) {
	......
	// 分配indexID,indexID=0
	indexID, err := s.meta.CanCreateIndex(req)
	......

	if indexID == 0 {
		// 分配indexID
		indexID, err = s.allocator.allocID(ctx)
		......
	}

	index := &model.Index{
		CollectionID:    req.GetCollectionID(),
		FieldID:         req.GetFieldID(),
		IndexID:         indexID,
		IndexName:       req.GetIndexName(),
		TypeParams:      req.GetTypeParams(),
		IndexParams:     req.GetIndexParams(),
		CreateTime:      req.GetTimestamp(),
		IsAutoIndex:     req.GetIsAutoIndex(),
		UserIndexParams: req.GetUserIndexParams(),
	}

	// Get flushed segments and create index

	err = s.meta.CreateIndex(index)
	......

	// 将collectionID发送到channel,其它的goroutine进行消费。
	select {
	case s.notifyIndexChan <- req.GetCollectionID():
	default:
	}

	......
}
```

![index调试](/milvus/index调试.jpg)

### 5.进入s.meta.CreateIndex()

代码路径:internal\datacoord\index_meta.go

```go
func (m *meta) CreateIndex(index *model.Index) error {
	......
	// 写入etcd元数据
	if err := m.catalog.CreateIndex(m.ctx, index); err != nil {
		......
	}

	m.updateCollectionIndex(index)
	......
}
```

在这里重点研究m.catalog.CreateIndex()这个方法做了什么事情。

```go
func (kc *Catalog) CreateIndex(ctx context.Context, index *model.Index) error {
	key := BuildIndexKey(index.CollectionID, index.IndexID)

	value, err := proto.Marshal(model.MarshalIndexModel(index))
	if err != nil {
		return err
	}

	err = kc.MetaKv.Save(key, string(value))
	if err != nil {
		return err
	}
	return nil
}
```

在etcd会产生1个key。

==field-index/445834678636119060/445834678636519085==

value的值的结构为indexpb.FieldIndex，然后进行protobuf序列化后存入etcd。

因此etcd存储的是二进制数据。

```go
&indexpb.FieldIndex{
	IndexInfo: &indexpb.IndexInfo{
		CollectionID:    index.CollectionID,
		FieldID:         index.FieldID,
		IndexName:       index.IndexName,
		IndexID:         index.IndexID,
		TypeParams:      index.TypeParams,
		IndexParams:     index.IndexParams,
		IsAutoIndex:     index.IsAutoIndex,
		UserIndexParams: index.UserIndexParams,
	},
	Deleted:    index.IsDeleted,
	CreateTime: index.CreateTime,
}
```

跟踪BuildIndexKey()函数，即可以得到key的规则。整理如下:

key规则:

- 前缀/field-index/{collectionID}/{IndexID}

可以反映index属于哪个collection。Index的value可以反映属于哪个field。

不能反映属于哪个partition、哪个segment。

总结:

- CreateIndex由proxy传递给协调器dataCoord操作etcd。
- CreateIndex最终会在etcd上写入1种类型的key（其实还有一种）。



## 创建索引操作

### 1.dataCoord执行CreateIndex

```go
func (s *Server) CreateIndex(ctx context.Context, req *indexpb.CreateIndexRequest) (*commonpb.Status, error) {
	......
	// 分配indexID,indexID=0
	indexID, err := s.meta.CanCreateIndex(req)
	......

	err = s.meta.CreateIndex(index)
	......

	// 将collectionID发送到channel,其它的goroutine进行消费。
	select {
	case s.notifyIndexChan <- req.GetCollectionID():
	default:
	}

	......
}
```

上文已经分析过s.meta.CreateIndex()，这里重点分析s.notifyIndexChan。将collectionID发送到channel,其它的goroutine进行消费。

### 2.消费notifyIndexChan也在dataCoord

代码路径:internal\datacoord\index_service.go

启动dataCoord服务的时候会调用此方法。

```go
func (s *Server) createIndexForSegmentLoop(ctx context.Context) {
	log.Info("start create index for segment loop...")
	defer s.serverLoopWg.Done()

	ticker := time.NewTicker(time.Minute)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			log.Warn("DataCoord context done, exit...")
			return
		case <-ticker.C:
			segments := s.meta.GetHasUnindexTaskSegments()
			for _, segment := range segments {
				if err := s.createIndexesForSegment(segment); err != nil {
					log.Warn("create index for segment fail, wait for retry", zap.Int64("segmentID", segment.ID))
					continue
				}
			}
		case collectionID := <-s.notifyIndexChan:
			log.Info("receive create index notify", zap.Int64("collectionID", collectionID))
			// 获取collection的segment信息
			segments := s.meta.SelectSegments(func(info *SegmentInfo) bool {
				return isFlush(info) && collectionID == info.CollectionID
			})
			for _, segment := range segments {
				if err := s.createIndexesForSegment(segment); err != nil {
					log.Warn("create index for segment fail, wait for retry", zap.Int64("segmentID", segment.ID))
					continue
				}
			}
		case segID := <-s.buildIndexCh:
			log.Info("receive new flushed segment", zap.Int64("segmentID", segID))
			segment := s.meta.GetSegment(segID)
			if segment == nil {
				log.Warn("segment is not exist, no need to build index", zap.Int64("segmentID", segID))
				continue
			}
			if err := s.createIndexesForSegment(segment); err != nil {
				log.Warn("create index for segment fail, wait for retry", zap.Int64("segmentID", segment.ID))
				continue
			}
		}
	}
}
```

走case collectionID := <-s.notifyIndexChan这个分支。

### 3.进入s.createIndexesForSegment()

```go
func (s *Server) createIndexesForSegment(segment *SegmentInfo) error {
	// 要创建的索引的信息：索引名称，维度等信息
	indexes := s.meta.GetIndexesForCollection(segment.CollectionID, "")
	for _, index := range indexes {
		if _, ok := segment.segmentIndexes[index.IndexID]; !ok {
			if err := s.createIndexForSegment(segment, index.IndexID); err != nil {
				log.Warn("create index for segment fail", zap.Int64("segmentID", segment.ID),
					zap.Int64("indexID", index.IndexID))
				return err
			}
		}
	}
	return nil
}
```

4.进入s.createIndexForSegment()

在segment上建立索引。

```go
func (s *Server) createIndexForSegment(segment *SegmentInfo, indexID UniqueID) error {
	log.Info("create index for segment", zap.Int64("segmentID", segment.ID), zap.Int64("indexID", indexID))
	buildID, err := s.allocator.allocID(context.Background())
	if err != nil {
		return err
	}
	segIndex := &model.SegmentIndex{
		SegmentID:    segment.ID,
		CollectionID: segment.CollectionID,
		PartitionID:  segment.PartitionID,
		NumRows:      segment.NumOfRows,
		IndexID:      indexID,
		BuildID:      buildID,
		CreateTime:   uint64(segment.ID),
		WriteHandoff: false,
	}
	// 添加segment-index
	// 前缀/segment-index/{collectionID}/{partitionID}/{segmentID}/{buildID}
	if err = s.meta.AddSegmentIndex(segIndex); err != nil {
		return err
	}
	// 入队，通知IndexNode执行索引创建任务
	s.indexBuilder.enqueue(buildID)
	return nil
}
```

s.meta.AddSegmentIndex(segIndex)在etcd添加segment-index。

==前缀/segment-index/{collectionID}/{partitionID}/{segmentID}/{buildID}==

```go
func (kc *Catalog) CreateSegmentIndex(ctx context.Context, segIdx *model.SegmentIndex) error {
    // key规则
	key := BuildSegmentIndexKey(segIdx.CollectionID, segIdx.PartitionID, segIdx.SegmentID, segIdx.BuildID)

	value, err := proto.Marshal(model.MarshalSegmentIndexModel(segIdx))
	if err != nil {
		return err
	}
    // 写入etcd
	err = kc.MetaKv.Save(key, string(value))
	if err != nil {
		log.Error("failed to save segment index meta in etcd", zap.Int64("buildID", segIdx.BuildID),
			zap.Int64("segmentID", segIdx.SegmentID), zap.Error(err))
		return err
	}
	return nil
}
```

### 5.进入s.indexBuilder.enqueue(buildID)

此函数作用:入队，通知IndexNode执行索引创建任务。

![数据流](/milvus/index数据流图.jpg)

### 6.进入process()

```go
func (ib *indexBuilder) process(buildID UniqueID) bool {
	......
	// 有一个定时器，默认1秒，配置项indexCoord.scheduler.interval
	switch state {
	case indexTaskInit:
		segment := ib.meta.GetSegment(meta.SegmentID)
		if !isSegmentHealthy(segment) || !ib.meta.IsIndexExist(meta.CollectionID, meta.IndexID) {
			log.Ctx(ib.ctx).Info("task is no need to build index, remove it", zap.Int64("buildID", buildID))
			if err := ib.meta.DeleteTask(buildID); err != nil {
				log.Ctx(ib.ctx).Warn("IndexCoord delete index failed", zap.Int64("buildID", buildID), zap.Error(err))
				return false
			}
			deleteFunc(buildID)
			return true
		}
		indexParams := ib.meta.GetIndexParams(meta.CollectionID, meta.IndexID)
		if isFlatIndex(getIndexType(indexParams)) || meta.NumRows < Params.DataCoordCfg.MinSegmentNumRowsToEnableIndex.GetAsInt64() {
			// 数量小于最低数量无需建立索引,默认为1024
			log.Ctx(ib.ctx).Info("segment does not need index really", zap.Int64("buildID", buildID),
				zap.Int64("segmentID", meta.SegmentID), zap.Int64("num rows", meta.NumRows))
			if err := ib.meta.FinishTask(&indexpb.IndexTaskInfo{
				BuildID:        buildID,
				State:          commonpb.IndexState_Finished,
				IndexFileKeys:  nil,
				SerializedSize: 0,
				FailReason:     "",
			}); err != nil {
				log.Ctx(ib.ctx).Warn("IndexCoord update index state fail", zap.Int64("buildID", buildID), zap.Error(err))
				return false
			}
			updateStateFunc(buildID, indexTaskDone)
			return true
		}
		// peek client
		// if all IndexNodes are executing task, wait for one of them to finish the task.
		nodeID, client := ib.nodeManager.PeekClient(meta)
		if client == nil {
			log.Ctx(ib.ctx).WithRateGroup("dc.indexBuilder", 1, 60).RatedInfo(5, "index builder peek client error, there is no available")
			return false
		}
		// update version and set nodeID
		if err := ib.meta.UpdateVersion(buildID, nodeID); err != nil {
			log.Ctx(ib.ctx).Warn("index builder update index version failed", zap.Int64("build", buildID), zap.Error(err))
			return false
		}
		// binLogs为向量数据文件的位置信息
		// files/insert_log/444517122896489678/444517122896489679/444517122896489694/102/444517122896236031
		// files/insert_log/{collectionID}/{partitionID}/{segmentID}/{fieldID}/444517122896236031
		binLogs := make([]string, 0)
		fieldID := ib.meta.GetFieldIDByIndexID(meta.CollectionID, meta.IndexID)
		for _, fieldBinLog := range segment.GetBinlogs() {
			if fieldBinLog.GetFieldID() == fieldID {
				for _, binLog := range fieldBinLog.GetBinlogs() {
					binLogs = append(binLogs, binLog.LogPath)
				}
				break
			}
		}

		typeParams := ib.meta.GetTypeParams(meta.CollectionID, meta.IndexID)

		var storageConfig *indexpb.StorageConfig
		// 获取元数据存储类型：minio
		if Params.CommonCfg.StorageType.GetValue() == "local" {
			storageConfig = &indexpb.StorageConfig{
				RootPath:    Params.LocalStorageCfg.Path.GetValue(),
				StorageType: Params.CommonCfg.StorageType.GetValue(),
			}
		} else {
			storageConfig = &indexpb.StorageConfig{
				Address:          Params.MinioCfg.Address.GetValue(),
				AccessKeyID:      Params.MinioCfg.AccessKeyID.GetValue(),
				SecretAccessKey:  Params.MinioCfg.SecretAccessKey.GetValue(),
				UseSSL:           Params.MinioCfg.UseSSL.GetAsBool(),
				BucketName:       Params.MinioCfg.BucketName.GetValue(),
				RootPath:         Params.MinioCfg.RootPath.GetValue(),
				UseIAM:           Params.MinioCfg.UseIAM.GetAsBool(),
				IAMEndpoint:      Params.MinioCfg.IAMEndpoint.GetValue(),
				StorageType:      Params.CommonCfg.StorageType.GetValue(),
				Region:           Params.MinioCfg.Region.GetValue(),
				UseVirtualHost:   Params.MinioCfg.UseVirtualHost.GetAsBool(),
				CloudProvider:    Params.MinioCfg.CloudProvider.GetValue(),
				RequestTimeoutMs: Params.MinioCfg.RequestTimeoutMs.GetAsInt64(),
			}
		}
		req := &indexpb.CreateJobRequest{
			ClusterID:           Params.CommonCfg.ClusterPrefix.GetValue(),
			IndexFilePrefix:     path.Join(ib.chunkManager.RootPath(), common.SegmentIndexPath),
			BuildID:             buildID,
			DataPaths:           binLogs,
			IndexVersion:        meta.IndexVersion + 1,
			StorageConfig:       storageConfig,
			IndexParams:         indexParams,
			TypeParams:          typeParams,
			NumRows:             meta.NumRows,
			CurrentIndexVersion: ib.indexEngineVersionManager.GetCurrentIndexEngineVersion(),
		}
		// 通知IndexNode创建索引
		if err := ib.assignTask(client, req); err != nil {
			......
		}
		......
		// update index meta state to InProgress
		// 更新索引状态
		if err := ib.meta.BuildIndex(buildID); err != nil {
			......
		}
		updateStateFunc(buildID, indexTaskInProgress)

	case indexTaskDone:
		......

	default:
		......
	}
	return true
}
```

ib.assignTask()通知indexNode创建索引。

### 7.进入ib.assignTask()

```go
// assignTask sends the index task to the IndexNode, it has a timeout interval, if the IndexNode doesn't respond within
// the interval, it is considered that the task sending failed.
func (ib *indexBuilder) assignTask(builderClient types.IndexNodeClient, req *indexpb.CreateJobRequest) error {
	......
    // rpc调用indexNode
	resp, err := builderClient.CreateJob(ctx, req)
	......
}
```

### 8.进入builderClient.CreateJob()

代码路径:internal\indexnode\indexnode_service.go

```go
func (i *IndexNode) CreateJob(ctx context.Context, req *indexpb.CreateJobRequest) (*commonpb.Status, error) {
	......
	task := &indexBuildTask{
		ident:          fmt.Sprintf("%s/%d", req.ClusterID, req.BuildID),
		ctx:            taskCtx,
		cancel:         taskCancel,
		BuildID:        req.GetBuildID(),
		ClusterID:      req.GetClusterID(),
		node:           i,
		req:            req,
		cm:             cm,
		nodeID:         i.GetNodeID(),
		tr:             timerecord.NewTimeRecorder(fmt.Sprintf("IndexBuildID: %d, ClusterID: %s", req.BuildID, req.ClusterID)),
		serializedSize: 0,
	}
	ret := merr.Success()
	if err := i.sched.IndexBuildQueue.Enqueue(task); err != nil {
		......
	}
	......
}
```

task执行逻辑:

```go
pipelines := []func(context.Context) error{t.Prepare, t.BuildIndex, t.SaveIndexFiles}
```

依次执行task的Prepare()、BuildIndex()、SaveIndexFiles()方法。

### 9.进入BuildIndex()

```go
func (it *indexBuildTask) BuildIndex(ctx context.Context) error {
	......
	var buildIndexInfo *indexcgowrapper.BuildIndexInfo
	buildIndexInfo, err = indexcgowrapper.NewBuildIndexInfo(it.req.GetStorageConfig())
	......
	err = buildIndexInfo.AppendFieldMetaInfo(it.collectionID, it.partitionID, it.segmentID, it.fieldID, it.fieldType)
	......

	err = buildIndexInfo.AppendIndexMetaInfo(it.req.IndexID, it.req.BuildID, it.req.IndexVersion)
	......

	err = buildIndexInfo.AppendBuildIndexParam(it.newIndexParams)
	......

	jsonIndexParams, err := json.Marshal(it.newIndexParams)
	......

	err = buildIndexInfo.AppendBuildTypeParam(it.newTypeParams)
	......

	for _, path := range it.req.GetDataPaths() {
		err = buildIndexInfo.AppendInsertFile(path)
		if err != nil {
			log.Ctx(ctx).Warn("append insert binlog path failed", zap.Error(err))
			return err
		}
	}

	it.currentIndexVersion = getCurrentIndexVersion(it.req.GetCurrentIndexVersion())
	if err := buildIndexInfo.AppendIndexEngineVersion(it.currentIndexVersion); err != nil {
		log.Ctx(ctx).Warn("append index engine version failed", zap.Error(err))
		return err
	}
	// cgo层调用CreateIndex()
    // it.index有个Upload()方法写入s3(SaveIndexFiles方法里调用it.index.Upload()写入s3)
	it.index, err = indexcgowrapper.CreateIndex(ctx, buildIndexInfo)
	if err != nil {
		......
	}

	......
}
```

indexcgowrapper.CreateIndex()调用C层创建索引。

### 10.进入SaveIndexFiles()

```go
func (it *indexBuildTask) SaveIndexFiles(ctx context.Context) error {
	......
	// c++层上传索引文件到s3
	indexFilePath2Size, err := it.index.UpLoad()
	......

}
```

总结:

- CreateIndex由proxy传递给协调器dataCoord操作etcd。
- 前缀/field-index/{collectionID}/{IndexID}
- 前缀/segment-index/{collectionID}/{partitionID}/{segmentID}/{buildID}
- dataCoord通知indexNode创建索引,indexNode通过cgo调用C++层创建索引，并上传至s3
