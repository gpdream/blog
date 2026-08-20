---
title: "lucene源码阅读3-flush"
date: "2018-05-04T08:13:33.000Z"
tags:
  - "es"
source_url: "https://gpdream.github.io/2018/05/04/es/lucene-3-flush/"
recovery_source: "html"
---
# lucene源码阅读3-flush

跟着debug走读flush全流程代码\

lucene在一些数据的写动作的时候，比如insert/update/delete等动作，并不会直接写入到磁盘，而是在内存中生效。然后再由某些条件去触发flush动作，这些动作有addIndexs,forceMerge,forceMergeDeletes,shutdown。还有在Insert和delete之类的动作，超过了LiveIndexWriterConfig.maxBufferedDocs和LiveIndexWriterConfig.ramBufferSizeMB之类的时候都会触发flush动作。当然也可以直接调用flush接口。

索引的flush入口在IndexWriter.flush。或者是上述的一些动作触发。`IndexWriter.flush`->`IndexWriter.doFLush()`.我们就从`doFlush()`方法开始看.代码看着比较多，但真正做flush动作的内容其实就是`docWriter.flushAllThreads();;`

```plain
private boolean doFlush(boolean applyAllDeletes) throws IOException {
   if (tragedy.get() != null) {
     throw new IllegalStateException("this writer hit an unrecoverable error; cannot flush", tragedy.get());
   }

   doBeforeFlush();
   testPoint("startDoFlush");
   boolean success = false;
   try {

     if (infoStream.isEnabled("IW")) {
       infoStream.message("IW", "  start flush: applyAllDeletes=" + applyAllDeletes);
       infoStream.message("IW", "  index before flush " + segString());
     }
     boolean anyChanges = false;

     synchronized (fullFlushLock) {
       boolean flushSuccess = false;
       try {
       //调用docWriter的flushALlThreads做flush动作
         long seqNo = docWriter.flushAllThreads();
         if (seqNo < 0) {
           seqNo = -seqNo;
           anyChanges = true;
         } else {
           anyChanges = false;
         }
         if (!anyChanges) {
           // flushCount is incremented in flushAllThreads
           flushCount.incrementAndGet();
         }
         flushSuccess = true;
       } finally {
         docWriter.finishFullFlush(this, flushSuccess);
         processEvents(false);
       }
     }

     if (applyAllDeletes) {
       applyAllDeletesAndUpdates();
     }

     anyChanges |= maybeMerge.getAndSet(false);

     synchronized(this) {
       doAfterFlush();
       success = true;
       return anyChanges;
     }
   } catch (VirtualMachineError tragedy) {
     tragicEvent(tragedy, "doFlush");
     throw tragedy;
   } finally {
     if (!success) {
       if (infoStream.isEnabled("IW")) {
         infoStream.message("IW", "hit exception during flush");
       }
       maybeCloseOnTragicEvent();
     }
   }
 }
```

很显然，直接看`DocWriter.flushAllThreads()`,这里做的事情，其实就把所有要flush的`DocumentsWriterPerThread`给拿出来全部轮一遍做`doFlush(flushingDWPT)`,flushingDWPT是`DocumentsWriterPerThread`类型，是一个代表索引文档的写线程对象。这个方法里才是做真正的flush。

```plain
long flushAllThreads()
   throws IOException {
   final DocumentsWriterDeleteQueue flushingDeleteQueue;
   if (infoStream.isEnabled("DW")) {
     infoStream.message("DW", "startFullFlush");
   }

   long seqNo;

   synchronized (this) {
     pendingChangesInCurrentFullFlush = anyChanges();
     flushingDeleteQueue = deleteQueue;
     /* Cutover to a new delete queue.  This must be synced on the flush control
      * otherwise a new DWPT could sneak into the loop with an already flushing
      * delete queue */
     seqNo = flushControl.markForFullFlush(); // swaps this.deleteQueue synced on FlushControl
     assert setFlushingDeleteQueue(flushingDeleteQueue);
   }
   assert currentFullFlushDelQueue != null;
   assert currentFullFlushDelQueue != deleteQueue;

   boolean anythingFlushed = false;
   try {
     DocumentsWriterPerThread flushingDWPT;
     // Help out with flushing:
     //把要flush的线程一个个拖出来给轮了
     while ((flushingDWPT = flushControl.nextPendingFlush()) != null) {
     //做具体的doFlush
       anythingFlushed |= doFlush(flushingDWPT);
     }
     // If a concurrent flush is still in flight wait for it
     flushControl.waitForFlush();
     if (anythingFlushed == false && flushingDeleteQueue.anyChanges()) { // apply deletes if we did not flush any document
       if (infoStream.isEnabled("DW")) {
         infoStream.message("DW", Thread.currentThread().getName() + ": flush naked frozen global deletes");
       }
       ticketQueue.addDeletes(flushingDeleteQueue);
     }
     ticketQueue.forcePurge(writer);
     // we can't assert that we don't have any tickets in teh queue since we might add a DocumentsWriterDeleteQueue
     // concurrently if we have very small ram buffers this happens quite frequently
     assert !flushingDeleteQueue.anyChanges();
   } finally {
     assert flushingDeleteQueue == currentFullFlushDelQueue;
   }
   if (anythingFlushed) {
     return -seqNo;
   } else {
     return seqNo;
   }
 }
```

由此为止，目前的调用链为`IndexWriter.doFLush()`->`DocWriter.flushAllThreads()`,`DocWriter.doFlush(flushingDWPT)`。贴`doFlush(flushingDWPT)的代码`。这里做的具体的动作有`ticket = ticketQueue.addFlushTicket(flushingDWPT)`;ticketQueue被定义为`DocumentsWriterFlushQueue`，用来同步多个flush线程。其中`ticketQueue.addFlushTicket(flushingDWPT)`做了`flushingDWPT.prepareFlush()`，这里是把一些被标记为删除的数据给提前删除掉了。然后做真正的flush动作`flushingDWPT.flush()`.在flush完成后把情况添加到ticketQueue中。

```plain
private boolean doFlush(DocumentsWriterPerThread flushingDWPT) throws IOException {
    boolean hasEvents = false;
    while (flushingDWPT != null) {
      hasEvents = true;
      boolean success = false;
      SegmentFlushTicket ticket = null;
      try {
        assert currentFullFlushDelQueue == null
            || flushingDWPT.deleteQueue == currentFullFlushDelQueue : "expected: "
            + currentFullFlushDelQueue + "but was: " + flushingDWPT.deleteQueue
            + " " + flushControl.isFullFlush();
        /*
         由于DWPT是并发可能有多个线程同时执行的，所以要保证flush的顺序，在flush segment的时候删除缓存。
         这是因为当取出一个DWPT的时候会标记一个全局删除掉点，当flush完的时候会删除这个点的缓存。
         举例有一个flush A启动并冻结全局删除，然后又有一个flush B启动并冻结自A以后所有的全局删除。如果B在A之前完成，我们需要等待A完成，否则B标记的删除可能不会被用于A，会错过被A标记删除的文档
         */
        try {
          // Each flush is assigned a ticket in the order they acquire the ticketQueue lock
          //这里是做准备工作，让flushingDWPT做准备工作并addTicket到队列中
          ticket = ticketQueue.addFlushTicket(flushingDWPT);
          final int flushingDocsInRam = flushingDWPT.getNumDocsInRAM();
          boolean dwptSuccess = false;
          try {
            // 这里是真正的segemnt flush
            final FlushedSegment newSegment = flushingDWPT.flush();
            //把新flush进去的segment添加到队列中
            ticketQueue.addSegment(ticket, newSegment);
            dwptSuccess = true;
          } finally {
            subtractFlushedNumDocs(flushingDocsInRam);
            if (flushingDWPT.pendingFilesToDelete().isEmpty() == false) {
              putEvent(new DeleteNewFilesEvent(flushingDWPT.pendingFilesToDelete()));
              hasEvents = true;
            }
            if (dwptSuccess == false) {
              putEvent(new FlushFailedEvent(flushingDWPT.getSegmentInfo()));
              hasEvents = true;
            }
          }
          // flush was successful once we reached this point - new seg. has been assigned to the ticket!
          success = true;
        } finally {
          if (!success && ticket != null) {
            // In the case of a failure make sure we are making progress and
            // apply all the deletes since the segment flush failed since the flush
            // ticket could hold global deletes see FlushTicket#canPublish()
            ticketQueue.markTicketFailed(ticket);
          }
        }
        /*
         * Now we are done and try to flush the ticket queue if the head of the
         * queue has already finished the flush.
         */
        if (ticketQueue.getTicketCount() >= perThreadPool.getActiveThreadStateCount()) {
          // This means there is a backlog: the one
          // thread in innerPurge can't keep up with all
          // other threads flushing segments.  In this case
          // we forcefully stall the producers.
          putEvent(ForcedPurgeEvent.INSTANCE);
          break;
        }
      } finally {
        flushControl.doAfterFlush(flushingDWPT);
      }

      //开始flush 下一个
      flushingDWPT = flushControl.nextPendingFlush();
    }

    if (hasEvents) {
      writer.doAfterSegmentFlushed(false, false);
    }

    // If deletes alone are consuming > 1/2 our RAM
    // buffer, force them all to apply now. This is to
    // prevent too-frequent flushing of a long tail of
    // tiny segments:
    final double ramBufferSizeMB = config.getRAMBufferSizeMB();
    if (ramBufferSizeMB != IndexWriterConfig.DISABLE_AUTO_FLUSH &&
        flushControl.getDeleteBytesUsed() > (1024*1024*ramBufferSizeMB/2)) {
      hasEvents = true;
      if (applyAllDeletes(deleteQueue) == false) {
        if (infoStream.isEnabled("DW")) {
          infoStream.message("DW", String.format(Locale.ROOT, "force apply deletes after flush bytesUsed=%.1f MB vs ramBuffer=%.1f MB",
                                                 flushControl.getDeleteBytesUsed()/(1024.*1024.),
                                                 ramBufferSizeMB));
        }
        putEvent(ApplyDeletesEvent.INSTANCE);
      }
    }

    return hasEvents;
  }
```

这里是`DocumentsWriterPerThread.flush()`,初始化了一个`SegmentWriteState flushState`,`flushState`中包含了该分片的所有信息，由 `consumer.flush(flushState);`完成把文件的刷入。下面重点看`consumer.flush(flushState)`

```plain
//把所有准备好的document 刷到一个新的segment
 FlushedSegment flush() throws IOException {
    assert numDocsInRAM > 0;
    //确保所有被标记为删除的数据在prepareFLush阶段被删除
    assert deleteSlice.isEmpty() : "all deletes must be applied in prepareFlush";
    //设置segmentInfo
    segmentInfo.setMaxDoc(numDocsInRAM);
    //初始化一个flushState，里面包含了所有要flush成一个segment的信息，后面consumer会根据这个去做flush
    final SegmentWriteState flushState = new SegmentWriteState(infoStream, directory, segmentInfo, fieldInfos.finish(),
        pendingUpdates, new IOContext(new FlushInfo(numDocsInRAM, bytesUsed())));
    final double startMBUsed = bytesUsed() / 1024. / 1024.;

    // Apply delete-by-docID now (delete-byDocID only
    // happens when an exception is hit processing that
    // doc, eg if analyzer has some problem w/ the text):
    if (pendingUpdates.deleteDocIDs.size() > 0) {
      flushState.liveDocs = codec.liveDocsFormat().newLiveDocs(numDocsInRAM);
      for(int delDocID : pendingUpdates.deleteDocIDs) {
        flushState.liveDocs.clear(delDocID);
      }
      flushState.delCountOnFlush = pendingUpdates.deleteDocIDs.size();
      pendingUpdates.bytesUsed.addAndGet(-pendingUpdates.deleteDocIDs.size() * BufferedUpdates.BYTES_PER_DEL_DOCID);
      pendingUpdates.deleteDocIDs.clear();
    }

    if (aborted) {
      if (infoStream.isEnabled("DWPT")) {
        infoStream.message("DWPT", "flush: skip because aborting is set");
      }
      return null;
    }

    long t0 = System.nanoTime();

    if (infoStream.isEnabled("DWPT")) {
      infoStream.message("DWPT", "flush postings as segment " + flushState.segmentInfo.name + " numDocs=" + numDocsInRAM);
    }
    final Sorter.DocMap sortMap;
    try {
      sortMap = consumer.flush(flushState);
      // We clear this here because we already resolved them (private to this segment) when writing postings:
      pendingUpdates.clearDeleteTerms();
      segmentInfo.setFiles(new HashSet<>(directory.getCreatedFiles()));

      final SegmentCommitInfo segmentInfoPerCommit = new SegmentCommitInfo(segmentInfo, 0, -1L, -1L, -1L);
      if (infoStream.isEnabled("DWPT")) {
        infoStream.message("DWPT", "new segment has " + (flushState.liveDocs == null ? 0 : flushState.delCountOnFlush) + " deleted docs");
        infoStream.message("DWPT", "new segment has " +
            (flushState.fieldInfos.hasVectors() ? "vectors" : "no vectors") + "; " +
            (flushState.fieldInfos.hasNorms() ? "norms" : "no norms") + "; " +
            (flushState.fieldInfos.hasDocValues() ? "docValues" : "no docValues") + "; " +
            (flushState.fieldInfos.hasProx() ? "prox" : "no prox") + "; " +
            (flushState.fieldInfos.hasFreq() ? "freqs" : "no freqs"));
        infoStream.message("DWPT", "flushedFiles=" + segmentInfoPerCommit.files());
        infoStream.message("DWPT", "flushed codec=" + codec);
      }

      final BufferedUpdates segmentDeletes;
      if (pendingUpdates.deleteQueries.isEmpty() && pendingUpdates.numericUpdates.isEmpty() && pendingUpdates.binaryUpdates.isEmpty()) {
        pendingUpdates.clear();
        segmentDeletes = null;
      } else {
        segmentDeletes = pendingUpdates;
      }

      if (infoStream.isEnabled("DWPT")) {
        final double newSegmentSize = segmentInfoPerCommit.sizeInBytes() / 1024. / 1024.;
        infoStream.message("DWPT", "flushed: segment=" + segmentInfo.name +
            " ramUsed=" + nf.format(startMBUsed) + " MB" +
            " newFlushedSize=" + nf.format(newSegmentSize) + " MB" +
            " docs/MB=" + nf.format(flushState.segmentInfo.maxDoc() / newSegmentSize));
      }

      assert segmentInfo != null;

      FlushedSegment fs = new FlushedSegment(infoStream, segmentInfoPerCommit, flushState.fieldInfos,
          segmentDeletes, flushState.liveDocs, flushState.delCountOnFlush,
          sortMap);
     //合并segment文件
      sealFlushedSegment(fs, sortMap);
      if (infoStream.isEnabled("DWPT")) {
        infoStream.message("DWPT", "flush time " + ((System.nanoTime() - t0) / 1000000.0) + " msec");
      }
      return fs;
    } catch (Throwable t) {
      onAbortingException(t);
      throw t;
    } finally {
      maybeAbort("flush");
    }
  }
```

`DefaultIndexingChain.flush(SegmentWriteState state)`是把文件刷入具体的磁盘。这里第一步是`writeNorms(state, sortMap);`会把segment的codec格式，索引头，字段域信息，字段规范等等信息写入到索引的`_32a.nvd`和`_32a.nvm`这两文件中。然后是`writeDocValues(state, sortMap);`把docValue写入到数据中,再然后是`writePoints(state, sortMap);`,把ponitValues(取代NumericField)写入，这些东西会被写到.dim文件中；接下来`storedFieldsConsumer.finish(maxDoc);storedFieldsConsumer.flush(state, sortMap);`会把数据写入到.fdt和.fdx中，这两个文件占了索引存储的大头；`termsHash.flush(fieldsToFlush, state, sortMap,normsMergeInstance);`把词元信息写入到.tvd和.tvx文件;`docWriter.codec.fieldInfosFormat().write(state.directory, state.segmentInfo, "", state.fieldInfos, IOContext.DEFAULT);`把数据写入.fnm程序。

```plain
@Override
  public Sorter.DocMap flush(SegmentWriteState state) throws IOException {

    // NOTE: caller (DocumentsWriterPerThread) handles
    // aborting on any exception from this method
    Sorter.DocMap sortMap = maybeSortSegment(state);
    int maxDoc = state.segmentInfo.maxDoc();
    long t0 = System.nanoTime();
    //写入规范，也就是codec格式，索引头，字段域信息，字段规范等等
    writeNorms(state, sortMap);
    if (docState.infoStream.isEnabled("IW")) {
      docState.infoStream.message("IW", ((System.nanoTime()-t0)/1000000) + " msec to write norms");
    }
    SegmentReadState readState = new SegmentReadState(state.directory, state.segmentInfo, state.fieldInfos, IOContext.READ, state.segmentSuffix);

    t0 = System.nanoTime();
    writeDocValues(state, sortMap);
    if (docState.infoStream.isEnabled("IW")) {
      docState.infoStream.message("IW", ((System.nanoTime()-t0)/1000000) + " msec to write docValues");
    }

    t0 = System.nanoTime();
    writePoints(state, sortMap);
    if (docState.infoStream.isEnabled("IW")) {
      docState.infoStream.message("IW", ((System.nanoTime()-t0)/1000000) + " msec to write points");
    }

    // it's possible all docs hit non-aborting exceptions...
    t0 = System.nanoTime();
    storedFieldsConsumer.finish(maxDoc);
    storedFieldsConsumer.flush(state, sortMap);
    if (docState.infoStream.isEnabled("IW")) {
      docState.infoStream.message("IW", ((System.nanoTime()-t0)/1000000) + " msec to finish stored fields");
    }

    t0 = System.nanoTime();
    Map<String,TermsHashPerField> fieldsToFlush = new HashMap<>();
    for (int i=0;i<fieldHash.length;i++) {
      PerField perField = fieldHash[i];
      while (perField != null) {
        if (perField.invertState != null) {
          fieldsToFlush.put(perField.fieldInfo.name, perField.termsHashPerField);
        }
        perField = perField.next;
      }
    }

    try (NormsProducer norms = readState.fieldInfos.hasNorms()
        ? state.segmentInfo.getCodec().normsFormat().normsProducer(readState)
        : null) {
      NormsProducer normsMergeInstance = null;
      if (norms != null) {
        // Use the merge instance in order to reuse the same IndexInput for all terms
        normsMergeInstance = norms.getMergeInstance();
      }
      //刷入词元
      termsHash.flush(fieldsToFlush, state, sortMap, normsMergeInstance);
    }
    if (docState.infoStream.isEnabled("IW")) {
      docState.infoStream.message("IW", ((System.nanoTime()-t0)/1000000) + " msec to write postings and finish vectors");
    }

    // Important to save after asking consumer to flush so
    // consumer can alter the FieldInfo* if necessary.  EG,
    // FreqProxTermsWriter does this with
    // FieldInfo.storePayload.
    t0 = System.nanoTime();
    docWriter.codec.fieldInfosFormat().write(state.directory, state.segmentInfo, "", state.fieldInfos, IOContext.DEFAULT);
    if (docState.infoStream.isEnabled("IW")) {
      docState.infoStream.message("IW", ((System.nanoTime()-t0)/1000000) + " msec to write fieldInfos");
    }

    return sortMap;
  }
```

到此为止，所有的flush动作结束。一个完成的flush流程为`IndexWriter.doFLush()`->`DocWriter.flushAllThreads()`->`DocWriter.doFlush(flushingDWPT)`->`DocumentsWriterPerThread.flush()`->`DefaultIndexingChain.flush(SegmentWriteState state)`。最后的`DefaultIndexingChain.flush(SegmentWriteState state)`会flush各种数据到lucene的索引文件中。在segment flush完成后，DocumentWriter会尝试合并segment。

### 索引segment下的文件格式

直接从网上拿了个文件格式对应的文件

| 文件名 | 后缀 | 描述 |
| --- | --- | --- |
| Segments File | segments.gen, segments_N | 存储段文件的提交点信息 |
| Lock File | write.lock | 文件锁，保证任何时刻只有一个线程可以写入索引 |
| Segment Info | .si | 存储每个段文件的元数据信息 |
| Compound File | .cfs, .cfe | 复合索引的文件，在系统上虚拟的一个文件，用于频繁的文件句柄 |
| Fields | .fnm | 存储域文件的信息 |
| Field Index | .fdx | 存储域数据的指针 |
| Field Data | .fdt | 存储所有文档的字段信息 |
| Term Dictionary | .tim | term字典，存储term信息 |
| Term Index | .tip | term字典的索引文件 |
| Frequencies | .frq | 词频文件，包含文档列表以及每一个term和其词频 |
| Positions | .prx | 位置信息，存储每个term，在索引中的准确位置 |
| Norms | .nrm.cfs, .nrm.cfe | 存储文档和域的编码长度以及加权因子 |
| Per-Document Values | .dv.cfs, .dv.cfe | 编码除外的额外的打分因素， |
| Term Vector Index | .tvx | term向量索引，存储term在文档中的偏移距离 |
| Term Vector Documents | .tvd | 包含每个文档向量的信息 |
| Term Vector Fields | .tvf | 存储filed级别的向量信息 |
| Deleted Documents | .del | 存储索引删除文件的信息 |
