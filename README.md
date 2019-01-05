## impala server的启动
### 启动脚本

*注： 此处的启动命令sudo service xx start 为手动将某个启动脚本加入为
服务，因此 impala-server 为服务名，并不是启动脚本的文件名*
````
//启动state store
$ sudo service impala-state-store start

//启动catalog-store
$ sudo service impala-catalog start

//启动impala-deamon 主服务，对应启动脚本为bin/start-impalad.sh
$ sudo service impala-server start

````
#### start-impalad.sh 
````
//启动参数
IMPALAD_BINARY=service/impalad
BINARY=${IMPALAD_BINARY}
IMPALA_CMD=${BINARY_BASE_DIR}/${BUILD_TYPE}/${BINARY}
//启动命令,args传递参数，例如启动几个daemon
exec ${TOOL_PREFIX} ${IMPALA_CMD} ${IMPALAD_ARGS}

````

### service/impalad
service/impalad 所指的是编译打包后的路径以及可执行文件。检查impala/be/src/service
所对应的CMAKELISTS.txt（编译打包文件）发现：
````
add_executable(impalad
  daemon-main.cc
)
````
#### daemon-main.cc
````
 if (daemon == "statestored") {
    return StatestoredMain(argc, argv);
  }

//启动impalad
  if (daemon == "impalad") {
  //实现方法对应service文件夹下impalad-main.cc
    return ImpaladMain(argc, argv);
  }

  if (daemon == "catalogd") {
    return CatalogdMain(argc, argv);
  }
````

#### impalad-main.cc
````
//对应service文件夹下impala-server.h/cc
impala_server->Start(FLAGS_be_port, FLAGS_beeswax_port(JDBC端口，HUE所用), FLAGS_hs2_port（(JDBC端口，其他)）);
````

#### impala-server
##### Status ImpalaServer::Start 方法

必须要有角色**TODO**
````
  if (!FLAGS_is_coordinator && !FLAGS_is_executor) {
    return Status("Impala does not have a valid role configured. "
        "Either --is_coordinator or --is_executor must be set to true.");
  }
````

impala 启动过程主要包括三块，backend(be)，BEESWAX，HS2.
角色不同，所需启动的服务也不同**TODO**
````
//通信服务
//用于deamon 之间通讯
ThriftServerBuilder be_builder("backend", be_processor, thrift_be_port);
//beeswax jdbc 通讯服务
ThriftServerBuilder builder(BEESWAX_SERVER_NAME_SERVER_NAME, beeswax_processor, beeswax_port);
//hive server 2通讯服务
ThriftServerBuilder builder(HS2_SERVER_NAME, hs2_fe_processor, hs2_port);

RETURN_IF_ERROR(
          builder.auth_provider(AuthManager::GetInstance()->GetExternalAuthProvider())
          .metrics(exec_env_->metrics())
          .max_concurrent_connections(FLAGS_fe_service_threads) //最大连接数，不填则无限制
          .Build(&server));
````


## impala SQL 的处理 （Beeswax 为例子）
````
boost::shared_ptr<TProcessor> hs2_fe_processor(
          //注册Thrift Service Processor
          new ImpalaHiveServer2ServiceProcessor(handler));
      boost::shared_ptr<TProcessorEventHandler> event_handler(
          //注册Thrift Service Handler
          new RpcEventHandler("hs2", exec_env_->metrics()));
      hs2_fe_processor->setEventHandler(event_handler);
````
### RpcEventHandler rpc-trace.cc

#### 1：解析请求数据，通过请求数据来调用thrift服务
````
 {
   "name": "beeswax",
   "methods": [
  {
     "name": "BeeswaxService.get_state",
     "summary": " count: 1, last: 0, min: 0, max: 0, mean: 0, stddev: 0",
     "in_flight": 0
     },
   {
     "name": "BeeswaxService.query",
     "summary": " count: 1, last: 293, min: 293, max: 293, mean: 293, stddev: 0",
     "in_flight": 0
     },
   ]
 }

````
#### 2：解析完请求后调用相对应的后端服务 //TODO
````
//通过请求调用后端服务
virtual void* getContext(const char* fn_name, void* server_context);
//找到对应的方法,lazay初始化 --> 如果没找到则加入此调用
it = method_map_.find(fn_name);
//调用方法环境
InvocationContext* ctxt_ptr =
      new InvocationContext(MonotonicMillis(), cnxn_ctx, it->second);
````

##### 2.1 BeeswaxService service/impala-beeswax-server.cc
````
//调用方法
void ImpalaServer::query(QueryHandle& query_handle, const Query& query){

//Execute(...) 开始处理 
RAISE_IF_ERROR(Execute(&query_ctx, session, &request_state),
        SQLSTATE_SYNTAX_ERROR_OR_ACCESS_VIOLATION);
}
````
##### 2.2 service/impala-server.cc
````
Status ImpalaServer::Execute(TQueryCtx* query_ctx,
    shared_ptr<SessionState> session_state,
    shared_ptr<ClientRequestState>* request_state){
    Status status = ExecuteInternal(*query_ctx, session_state, &registered_request_state,
          request_state);
    }

Status ImpalaServer::ExecuteInternal(
    const TQueryCtx& query_ctx,
    shared_ptr<SessionState> session_state,
    bool* registered_request_state,
    shared_ptr<ClientRequestState>* request_state) 
````

##### 2.3 ExecuteInternal
* 创建Request
````
//调用GetExecRequest
RETURN_IF_ERROR((*request_state)->UpdateQueryStatus(
        exec_env_->frontend()->GetExecRequest(query_ctx, &result)));

//service/frontend.cc -> GetExecRequest()
Status Frontend::GetExecRequest(
    const TQueryCtx& query_ctx, TExecRequest* result) {
    //通过JNI调用front end java 方法.
    //jmethodID create_exec_request_id_; JniFrontend.createExecRequest()
  return JniUtil::CallJniMethod(fe_, create_exec_request_id_, query_ctx, result);
}
````
````
JniFrontend.createExecRequest()
//返回TExecRequest
TExecRequest result = frontend_.createExecRequest(queryCtx, explainString);
````

JniFrontend.java -> createExecRequest(queryCtx, explainString);
````
public TExecRequest createExecRequest(TQueryCtx queryCtx, StringBuilder explainString)
      throws ImpalaException{
      //Frontend.java -> createExecRequest
      TExecRequest result = frontend_.createExecRequest(queryCtx, explainString);s
      }
````

Frontend.java -> createExecRequest //TODO StmtTableCache
````
//分析statement类型,例如：Create, Drop, Insert等等,稍后将以此为根据对sql进性分析
StatementBase stmt = parse(queryCtx.client_request.stmt);
//读取相应的tables的meta data，此数据将作为分信息数据被analyzer所使用
StmtTableCache stmtTableCache = metadataLoader.loadTables(stmt);
//开始分析query
AnalysisResult analysisResult =
        analysisCtx.analyzeAndAuthorize(stmt, stmtTableCache, authzChecker_.get());
````

AnalysisContext.java -> analyzeAndAuthorize(....);
````
analyzeAndAuthorize(....){
//分析
analyze(stmtTableCache);
}
````

AnalysisContext.java -> analyze()
````
//创建analyzer
analysisResult_.analyzer_ = createAnalyzer(stmtTableCache);
//根据不同类型的statement 调用不同的analyze方法
analysisResult_.stmt_.analyze(analysisResult_.analyzer_);
````

以 SelectStmt 类型为例
SelectStmt -> analyze(Analyzer analyzer)
````
// Start out with table refs to establish aliases.
//分析SQL中所用到的table： 
// 1.判断table 的种类,具体参照TableRef的子类，这里以BaseTableRef(HDFS/HDFS table)为例
    fromClause_.analyze(analyzer);
// 2.根据具体的table ref种类进行进一步的analyze,例如在BaseTableRef中有
    analyzeTableSample(analyzer);
     //验证TABLESAMPLE的有效性,不能大于100%或小于0%。 TABLESAMPLE的用法可参照Impala官方文档
     
    analyzeHints(analyzer);
    //impala 里有各种hints可以来指定例如join的顺序，这次检测了hint的使用，例如只能对HDFS table加hint
    //cross join不能使用shuffle等检测
    
    analyzeJoin(analyzer);
    analyzeSkipHeaderLineCount();
````



TExecRequest common/thrift/Frontend.thrift/TExecRequest
````
struct TExecRequest {
  1: required Types.TStmtType stmt_type

  // Copied from the corresponding TClientRequest
  2: required ImpalaInternalService.TQueryOptions query_options

  // TQueryExecRequest for the backend
  // Set iff stmt_type is QUERY or DML
  3: optional TQueryExecRequest query_exec_request

  // Set if stmt_type is DDL
  4: optional TCatalogOpRequest catalog_op_request

  // Metadata of the query result set (not set for DML)
  5: optional Results.TResultSetMetadata result_set_metadata

  // Result of EXPLAIN. Set iff stmt_type is EXPLAIN
  6: optional TExplainResult explain_result

  // Request for LOAD DATA statements.
  7: optional TLoadDataReq load_data_request

  // List of catalog objects accessed by this request. May be empty in this
  // case that the query did not access any Catalog objects.
  8: optional set<TAccessEvent> access_events

  // List of warnings that were generated during analysis. May be empty.
  9: required list<string> analysis_warnings

  // Set if stmt_type is SET
  10: optional TSetQueryOptionRequest set_query_option_request

  // Timeline of planner's operation, for profiling
  11: optional RuntimeProfile.TEventSequence timeline

  // If false, the user that runs this statement doesn't have access to the runtime
  // profile. For example, a user can't access the runtime profile of a query
  // that has a view for which the user doesn't have access to the underlying tables.
  12: optional bool user_has_profile_access
}
````
