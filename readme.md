# scalikesolr : Apache Solr Client for Scala/Java

"scalikesolr" is a simple Solr client library for [Scala 2.9.0.final](http://www.scala-lang.org/). And it also works fine with Java.

## How to install

### Maven

"3.1" means Solr/Lucene release version that is supported by scalikesolr. The "x" of "3.1.x" will be incremented.

    <repositories>
      <repository>
        <id>ScalikeSolrClientLibraryReleases</id>
        <url>https://github.com/seratch/scalikesolr/raw/master/mvn-repo/releases</url>
      </repository>
      <repository>
        <id>ScalikeSolrClientLibrarySnapshots</id>
        <url>https://github.com/seratch/scalikesolr/raw/master/mvn-repo/snapshots</url>
      </repository>
      ...
    </repositories>

    <dependency>
      <groupId>com.github.seratch</groupId>
      <artifactId>scalikesolr</artifactId>
      <version>[3.1,)</version>
    </dependency>

## Usage

### Query

#### Simple Query

Using [Core Query Paramters](http://wiki.apache.org/solr/CoreQueryParameters) and [Common Query Parameters](http://wiki.apache.org/solr/CommonQueryParameters):

    import com.github.seratch.scalikesolr._

    val client = Solr.httpServer(new URL("http://localhost:8983/solr")).newClient
    val request = new QueryRequest(Query("author:Rick"))
    val response = client.doQuery(request)
    println(response.responseHeader)
    println(response.response)
    response.response.documents foreach {
      case doc => {
        println(doc.get("id").toString()) // "978-1423103349"
        println(doc.get("cat").toListOrElse(Nil).toString) // List(book, hardcover)
        println(doc.get("title").toString()) // "The Lightning Thief"
        println(doc.get("pages_i").toIntOrElse(0).toString) // 384
        println(doc.get("price").toDoubleOrElse(0.0).toString) // 12.5
      }
    }

#### With Highlightings

Using [Highlighting Parameters](http://wiki.apache.org/solr/HighlightingParameters):

    val request = new QueryRequest(
      writerType = WriterType.JSON,
      query = Query("author:Rick"),
      sort = Sort("page_i desc"),
      highlighting = HighlightingParams(true)
    )
    val response = client.doQuery(request)
    println(response.highlightings)
    response.highlightings.keys foreach {
      case key => {
        println(key + " -> " + response.highlightings.get(key).get("author").toString)
        // "978-0641723445" -> "<em>Rick</em> Riordan"
      }
    }

#### With MoreLikeThis

Using [More Like This](http://wiki.apache.org/solr/MoreLikeThis):

    val request = new QueryRequest(
      query = Query("author:Rick"),
      moreLikeThis = MoreLikeThisParams(
        enabled = true,
        count = 3,
        fieldsToUseForSimilarity = FieldsToUseForSimilarity("body")
      )
    )
    val response = client.doQuery(request)
    println(response.moreLikeThis)
    response.response.documents foreach {
      doc => {
        val id = doc.get("id").toString
        println(id + " -> " + response.moreLikeThis.get(id).toString)
        // "978-0641723445" -> "SolrDocument(WriterType(standard),,Map(start -> 0, numFound -> 0))"
      }
    }

#### With FacetQuery

Using [Simple Facet Parameters](http://wiki.apache.org/solr/SimpleFacetParameters):

    val request = new QueryRequest(
      query = Query("author:Rick"),
      facet = new FacetParams(
        enabled = true,
        params = List(new FacetParam(Param("facet.field"), Value("title")))
      )
    )
    val response = client.doQuery(request)
    println(response.facet.facetFields)
    response.facet.facetFields.keys foreach {
      case key => {
        val facets = response.facet.facetFields.getOrElse(key, new SolrDocument())
        facets.keys foreach {
          case facetKey => println(facetKey + " -> " + facets.get(facetKey).toIntOrElse(0))
          // "thief" -> 1, "sea" -> 1, "monster" -> 1, "lightn" -> 1
        }
      }
    }

### DIH Command

Commands for [Data Import Handler](http://wiki.apache.org/solr/DataImportHandler):

    val request = new DIHCommandRequest(command = "delta-import")
    val response = client.doDIHCommand(request)
    println(response.initArgs)
    println(response.command)
    println(response.status)
    println(response.importResponse)
    println(response.statusMessages)

### Update

[XML Messages for Updating a Solr Index](http://wiki.apache.org/solr/UpdateXmlMessages):

#### Add documents

Add documents to Solr:

     val request = new AddRequest()
     val doc1 = SolrDocument(
       writerType = WriterType.JSON,
       rawBody = """
       {"id" : "978-0641723445",
        "cat" : ["book","hardcover"],
        "title" : "The Lightning Thief",
        "author" : "Rick Riordan",
        "series_t" : "Percy Jackson and the Olympians",
        "sequence_i" : 1,
        "genre_s" : "fantasy",
        "inStock" : true,
        "price" : 12.50,
        "pages_i" : 384
      }
      """
     )
     val doc2 = SolrDocument(
       writerType = WriterType.JSON,
       rawBody = """
       {"id" : "978-1423103349",
        "cat" : ["book","paperback"],
        "title" : "The Sea of Monsters",
        "author" : "Rick Riordan",
        "series_t" : "Percy Jackson and the Olympians",
        "sequence_i" : 2,
        "genre_s" : "fantasy",
        "inStock" : true,
        "price" : 6.49,
        "pages_i" : 304
       }
     """
     )
     request.documents = List(doc1, doc2)
     val response = client.doAddDocuments(request)
     client.doCommit(new UpdateRequest)

#### Delete documents

     val request = new DeleteRequest(uniqueKeysToDelete = List("978-0641723445"))
     val response = client.doDeleteDocuments(request)
     client.doCommit(new UpdateRequest)

#### Commit

     val response = client.doCommit(new UpdateRequest())

#### Rollback

     val response = client.doRollback(new UpdateRequest())

#### Optimize

     val response = client.doOptimize(new UpdateRequest())

### Ping

     val response = client.doPing(new PingRequest())
     println(response.status) // "OK"

### Loading documents from update format

JSON format is not supported.

#### [XML format](http://wiki.apache.org/solr/UpdateXmlMessages)

     val xmlString = "<add><doc><field name=\"employeeId\">05991</field><field name=\"office\">Bridgewater</field>..."
     val docs = UpdateFormatLoader.fromXMLString(xmlString)
     docs foreach {
       case doc => {
         println("employeeId:" + doc.get("employeeId").toString()) // "05991"
         println("office:" + doc.get("office").toString()) // "Bridgewater"
       }
     }

#### [CSV format](http://wiki.apache.org/solr/UpdateCSV)

     val csvString = "id,name,sequence_i\n0553573403,A Game of Thrones,1\n..."
     val docs = UpdateFormatLoader.fromCSVString(csvString)
     docs foreach {
       case doc => {
         println(doc.get("id")) // "0553573403"
         println(doc.get("name")) // "A Game of Thrones"
         println(doc.get("sequence_i").toIntOrElse(0)) // 1
       }
     }

### How to know which type is the param mapped?

Name of constructor arg is same as the Solr parameter key.

For example:

    case class Query(@BeanProperty val q: String) extends RequestParam

## Using in Java

This library works fine with Java.

### Query

    SolrClient client = Solr.httpServer(new URL("http://localhost:8983/solr")).getNewClient();
    QueryRequest request = new QueryRequest(Query.as("author:Rick")); // or new Query("author:Rick")
    request.setWriterType(WriterType.JSON());
    request.setSort(Sort.as("author desc"));
    request.setMoreLikeThis(MoreLikeThisParams.as(true, 3, FieldsToUseForSimilarity.as("title")));
    QueryResponse response = client.doQuery(request);
    List<SolrDocument> documents = response.getResponse().getDocumentsInJava();
    for (SolrDocument document : documents) {
        log.debug(document.get("id").toString()); // "978-1423103349"
        log.debug(document.get("cat").toListInJavaOrElse(null).toString()); // ["book", "paperback"]
        log.debug(document.get("pages_i").toNullableIntOrElse(null).toString()); // 304
        log.debug(document.get("price").toNullableDoubleOrElse(null).toString()); // 6.49
    }

### DIH Command

    DIHCommandRequest request = new DIHCommandRequest("delta-import");
    DIHCommandResponse response = client.doDIHCommand(request);

### Update

#### Add documements

    AddRequest request = new AddRequest();
    String jsonString = "{\"id\" : \"978-0641723445\", ... }";
    SolrDocument doc = new SolrDocument(WriterType.JSON(), jsonString);
    List<SolrDocument> docs = new ArrayList<SolrDocument>();
    docs.add(doc);
    request.setDocumentsInJava(docs);
    AddResponse response = client.doAddDocuments(request);
    client.doCommit(new UpdateRequest());

#### Delete documents

    DeleteRequest request = new DeleteRequest();
    List<String> uniqueKeys = new ArrayList<String>();
    uniqueKeys.add("978-0641723445");
    request.setUniqueKeysToDetelteInJava(uniqueKeys);
    DeleteResponse response = client.doDeleteDocuments(request);
    client.doCommit(new UpdateRequest());

#### Commit

    UpdateResponse response = client.doCommit(new UpdateRequest());

#### Rollback

    UpdateResponse response = client.doRollback(new UpdateRequest());

#### Optimize

    UpdateResponse response = client.doOptimize(new UpdateRequest());

### Ping

    PingResponse response = client.doPing(new PingRequest());

