##Brief Introduction of MySQL's performance_schema
The MySQL ***Performance Schema*** (usually referred to as ***PS***) is a feature for monitoring MySQL Server execution at a low level. 
<br/><br/>
This feature was first introduced in MySQL 5.5, become matured in 5.6, In MySQL 5.7, more features were introduced, also included a new schema named ***sys***, which is originated from [mysql-sys](https://github.com/mysql/mysql-sys) and aimed to make performance_schema easier to use. 
<br/><br/>
Why is ***PS*** so important, I want to borrow a paragraph from Marco Tusa in [his blog](http://www.pythian.com/blog/part-1-how-to-effectively-use-a-performance-schema/):
> Performance Schema(PS) has been the subject of many, many recent discussions, presentations, and
articles. After its release in MySQL 5.7, PS has become the main actor for people who want to take the further steps in MySQL monitoring. At the same time, it has become clear that Oracle intends to make PS powerful with so many features and new instrumentation that old-style monitoring will begin to look like obsolete tools from the Stone Age.

Briefly saying, MySQL's implementation of ***PS*** include two parts:

1. The `PERFORMANCE_SCHEMA` storage engine collects event data using "instrumentation points" in server source code.
2. Collected events are stored in tables in the `performance_schema` database. These tables can be queried using `SELECT` statements like other tables.

After checked with MySQL InnoDB developers, we confirmed that `5-10%` performance degradation will incur after turning on ***PS*** feature w/ common used instrumentation points in most production systems.
<br/><br/>
For more details about MySQL ***PS***, please refer to MySQL 5.7 Reference Manual [1-2].

##Ideas of TiDB's performance_schema
Generally saying, we want to port MySQL ***PS*** feature to TiDB, but we'are not going to replicate it, too many implementation related details inside, we are going to implement TiDB ***PS*** feature step by step.
<br/><br/>
Like MySQL, we have several design constraints:

* *The parser is unchanged*. Also, there are no keywords or statements. This guarantees that existing applications will run the same way with or without the Performance Schema.
* *All the instrumentation points return "void", there are no error codes*. Even if the performance shcema fails internally, execution of the server code will proceed.
* *None of the instrumentation points allocate memory*. All the memory used by the Performance Schema is preallocated at startup, and is considered "static" during the server life time.
* *None of the instrumentation points use any pthread_mutex, pthread_rwlock, or pthread_cond (or platform equivalents)*. Executing the instrumentation point should not cause thread scheduling to change in the server.

Because TiDB is a stateless SQL engine similar to Google F1, we also several other design constraints:

* The performance statistics of each TiDB server is independent, there has no relationship between them, and statistics data won't be replicated.
* The performance statistics of each TiDB server is runtime, thus that means the ***PS*** framework will not try to persist any historical data (of coz user can proactively use clauses like `INSERT INTO ... SELECT ...` to persist data), that is to say, each time after TiDB server start, the `performance_schema` is empty.
* Since TiDB server is written in Golang, which is lockless and depends on goroutines, all **wait** related event (mutex/rwlock/sxlock/cond) and **thread** related events will not occur in TiDB ***PS*** framework.

Also, from the implementation perspective, even TiDB can support multiple storage engines, but can't support different tables for different storage engines, so we will introduce a `performance_schema` database, but won't introduce a `PERFORMANCE_SCHEMA` storage engine.

##Implementation of TiDB's performance_schema

###3-Free Fules
The implementation needs to follow the 3-free rules:

* **Malloc** free
* **Mutex** free
* **Rwlock** free

###Setup Tables
Like MySQL, we also have several setup tables:

* **setup_actors**: defines which users will be monitored/instrumented
* **setup_objects**: which objects need to be monitored/instrumented
* **setup_instruments**: stores which monitoring/instrumentation points are enabled
* **setup_consumers**: stores which aggregation tables are maintained
* **setup_timers**: stores the unit of measurement of each instrumentation component

The initialization values of these setup tables are also similar to MySQL 5.7 except:

* TiDB does not have `PROCEDURE/TRIGGER` objects, also does not imeplement process tables in `information_schema` yet
* TiDB has 3 instrumentation components so far:
	* **Stage**: An instrumented stage event.
	* **Statement**: An instrumented statemenet event.
	* **Transaction**: An instrumented transaction event. This instrument has no further components.
* TiDB does not have `CYCLE` or `TICK` timer type since we have no **wait** or **thread** related events.

###Statement Event Tables
So far, TiDB has the following tables related to statement events:

* **events\_statements\_current**: Current statement events
* **events\_statements\_history**: The most recent statement events for each connection
* **events\_statements\_history\_long**: The most recent statement events overall
* **prepared\_statements_instances**: Prepared statement instances and statistics

Summary tables related to statement events will not be implemented so far.

###Transaction Event Tables
So far, TiDB has the following tables related to transaction events:

* **events\_transactions\_current**: Current transaction events
* **events\_transactions\_history**: The most recent transaction events for each connection
* **events\_transactions\_history\_long**: The most recent transaction events overall

Summary tables related to transaction events will not be implemented so far.

###Stage Event Tables
So far, TiDB has the follwing tables related to stage events:

* **events\_stages\_current**: Current stage events
* **events\_stages\_history**: The most recent stage events for each connection
* **events\_stages\_history\_long**: The most recent stage events overall

Summary tables related to stage events will not be implemented so far.

###Dependent Components
* [rcrowley/go-metrics](https://github.com/rcrowley/go-metrics): we use this library to implement **Instrumentation Point**.
* [syndtr/goleveldb](https://github.com/syndtr/goleveldb): this is the default storage engine of TiDB, but since TiDB can't support multiple storage engines at the same time, so our framework may need to invoke its key-value interfaces directly, *goleveldb* will be used in memory mode in order not to persist any data.

###Tools Compatibility
We would like to make TiDB's performance_schema to be compatible w/ MySQL's state-of-the-art tools such as MySQL Workbench, however, much of MySQL's instrumentation points are related to InnoDB storage engine, this is quite different from TiDB's, thus 100% compatibility is impossible.
<br/><br/>
So far, all tables in TiDB `performance_schema` can find an equivalent in MySQL, and all fields are the same.

###Special Notes
* Many tables in MySQL `performance_schema` has the field `THREAD_ID`, but TiDB is written in Go, and goroutines is much lighter than thread, and is not that fixed (one thread for one connection), then, we instead put connection id into the `THREAD_ID` field, but the field name will still be kept for compatibility concerns.

##References
[1] [MySQL 5.7 Reference Manual  /  MySQL Performance Schema](http://dev.mysql.com/doc/refman/5.7/en/performance-schema.html)
<br/>
[2] [MySQL 5.7 Reference Manual  /  MySQL sys Schema](http://dev.mysql.com/doc/refman/5.7/en/sys-schema.html)
