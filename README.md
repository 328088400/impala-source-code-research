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
````
