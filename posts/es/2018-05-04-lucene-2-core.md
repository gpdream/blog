---
title: "lucene源码阅读2-core部分-创建索引"
date: "2018-05-04T08:13:33.000Z"
tags:
  - "es"
source_url: "https://gpdream.github.io/2018/05/04/es/lucene-2-core/"
recovery_source: "html"
---
# lucene源码阅读2-core部分-创建索引

主要介绍lucene创建索引的源码部分\

## core主要包功能

- analysis: 分析器，带分词器过滤器之类的
- codes: 编码器
- document: lucene的document,包含filed,type之类的封装
- geo: geo地理位置信息
- index: IndexWriter和IndexReader之类写索引读索引的东西
- search: Query的数据结构,TermQuery,rangeQuery之类的
- store: 各种存储相关的，RamDirectory,FSDirectory之类的。
- util: 工具包

## 生成索引部分代码阅读

简单粗暴直接Debug运行单测TestIndexWriter.testDocCount()。首先看类`org.apache.lucene.index.IndexWriter`类。

### IndexWriter初始化索引

IndexWriter类，主要是提供了一些对索引的操作，比如对索引的close(),flush(),merge(),addDoc(),deleteAll()等索引的写操作。IndexWriter中主要有如下组件:

- analyzer: 预发分析器，带分词器和过滤器等
- codec: 编码器
- commitLock: 提交所
- deleter: IndexFileDeleter,用于删除segment文件，回滚commit的内容以及deleteAll()
- docWriter: DocumentsWriter，用于update,add,delete document等写操作
- eventQueue: Queue ,是docWriter的events队列，里面有`ApplyDeletesEvent`,`DeleteNewFielsEvent`,`FlushFailedEvent`,`ForcesPurgeEvent`,`ResolveUpdatesEvent`等事件.
- segmentInfos: segment集合的信息

除了初始化这些东西外,创建writer的时候还会CREATE_OR_APPEND指定的IndexWriter目录。

### IndexWriter写入文档

写入文档的接口为`IndexWriter.addDocument(Iterable<? extends IndexableField> doc)`.这个方法其实就是调用`DocumentWriter.updateDocument(final Iterable<? extends IndexableField> doc, final Analyzer analyzer,final DocumentsWriterDeleteQueue.Node<?> delNode)`方法，最后真正执行写入的是`DocumentsWriterPerThread.updateDocument(Iterable<? extends IndexableField> doc, Analyzer analyzer, DocumentsWriterDeleteQueue.Node<?> deleteNode)`.上源码

```plain
public long updateDocument(Iterable<? extends IndexableField> doc, Analyzer analyzer, DocumentsWriterDeleteQueue.Node<?> deleteNode) throws IOException {
    try {
      assert hasHitAbortingException() == false: "DWPT has hit aborting exception but is still indexing";
      testPoint("DocumentsWriterPerThread addDocument start");
      assert deleteQueue != null;
      reserveOneDoc();

      //初始化docState
      docState.doc = doc;
      docState.analyzer = analyzer;
      docState.docID = numDocsInRAM;
      if (INFO_VERBOSE && infoStream.isEnabled("DWPT")) {
        infoStream.message("DWPT", Thread.currentThread().getName() + " update delTerm=" + deleteNode + " docID=" + docState.docID + " seg=" + segmentInfo.name);
      }
      // Even on exception, the document is still added (but marked
      // deleted), so we don't need to un-reserve at that point.
      // Aborting exceptions will actually "lose" more than one
      // document, so the counter will be "wrong" in that case, but
      // it's very hard to fix (we can't easily distinguish aborting
      // vs non-aborting exceptions):
      boolean success = false;
      try {
        try {
        //由cosumer真正处理掉文档
          consumer.processDocument();
        } finally {
        //处理完清理掉docState,把docState.analyze=null,docState.doc=null
          docState.clear();
        }
        success = true;
      } finally {
      //不成功则回滚，删除掉doc
        if (!success) {
          // mark document as deleted
          deleteDocID(docState.docID);
          numDocsInRAM++;
        }
      }

      return finishDocument(deleteNode);
    } finally {
      maybeAbort("updateDocument");
    }
  }
```

至此完成了所有的写入文档。但是具体的索引倒排索引创建出来的逻辑还藏在`consumer.processDocument()`,顺便看看这里的逻辑，倒排索引的主要创建逻辑再`processField()`里。

````plain
 public void processDocument() throws IOException {

    // How many indexed field names we've seen (collapses
    // multiple field instances by the same name):
    int fieldCount = 0;

    long fieldGen = nextFieldGen++;

    // NOTE: we need two passes here, in case there are
    // multi-valued fields, because we must process all
    // instances of a given field at once, since the
    // analyzer is free to reuse TokenStream across fields
    // (i.e., we cannot have more than one TokenStream
    // running "at once"):

    termsHash.startDocument();

    startStoredFields(docState.docID);
    try {
    //把field拿出来一个个处理,field包含字段名，字段值,字段类型,是否存储，过滤器等等
      for (IndexableField field : docState.doc) {
        fieldCount = processField(field, fieldGen, fieldCount);
      }
    } finally {
      if (docWriter.hasHitAbortingException() == false) {
        // Finish each indexed field name seen in the document:
        for (int i=0;i<fieldCount;i++) {
          fields[i].finish();
        }
        finishStoredFields();
      }
    }

    try {
      termsHash.finishDocument();
    } catch (Throwable th) {
      // Must abort, on the possibility that on-disk term
      // vectors are now corrupt:
      docWriter.onAbortingException(th);
      throw th;
    }
  }
```

看看processField()的源码
````

private int processField(IndexableField field, long fieldGen, int fieldCount) throws IOException {\
 String fieldName = field.name();\
 IndexableFieldType fieldType = field.fieldType();

```
PerField fp = null;

if (fieldType.indexOptions() == null) {
  throw new NullPointerException("IndexOptions must not be null (field: \"" + field.name() + "\")");
}

//给字段建立倒排索引
// Invert indexed fields:
if (fieldType.indexOptions() != IndexOptions.NONE) {
//看字段是否存在
  fp = getOrAddField(fieldName, fieldType, true);
  boolean first = fp.fieldGen != fieldGen;
  //给这个建立对应的倒排索引，具体怎么建立倒排索引的下集讲
  fp.invert(field, first);

  if (first) {
    fields[fieldCount++] = fp;
    fp.fieldGen = fieldGen;
  }
} else {
  verifyUnIndexedFieldType(fieldName, fieldType);
}

// Add stored fields:
if (fieldType.stored()) {
  if (fp == null) {
    fp = getOrAddField(fieldName, fieldType, false);
  }
  if (fieldType.stored()) {
    String value = field.stringValue();
    if (value != null && value.length() > IndexWriter.MAX_STORED_STRING_LENGTH) {
      throw new IllegalArgumentException("stored field \"" + field.name() + "\" is too large (" + value.length() + " characters) to store");
    }
    try {
      storedFieldsConsumer.writeField(fp.fieldInfo, field);
    } catch (Throwable th) {
      docWriter.onAbortingException(th);
      throw th;
    }
  }
}

DocValuesType dvType = fieldType.docValuesType();
if (dvType == null) {
  throw new NullPointerException("docValuesType must not be null (field: \"" + fieldName + "\")");
}
if (dvType != DocValuesType.NONE) {
  if (fp == null) {
    fp = getOrAddField(fieldName, fieldType, false);
  }
  indexDocValue(fp, dvType, field);
}
if (fieldType.pointDimensionCount() != 0) {
  if (fp == null) {
    fp = getOrAddField(fieldName, fieldType, false);
  }
  indexPoint(fp, field);
}

return fieldCount;
```

}\

注意，上述的修改，都是在内存中建立的，并没有真正的刷新到内存中。IndexWriter会定期把内存中的数据(add,delete的数据)flush到内存中。当然，addIndex,forceMerge,deleteAll等动作也会强制触发flush,或者commit动作也会触发flush动作。

下集将flush，以后讲倒排索引是怎么建立起来的
