+++
title = "Benchmarks"
weight = 4
+++

In the past, many initial attempts at marrying a graph database with static analysis was done with TinkerGraph or Neo4j (see [Related Work](_index.md/#related-work)), however these databases have been proven to be slow and inefficient. This was motivation enough to create [OverfowDB](https://github.com/ShiftLeftSecurity/overflowdb), and now [flatgraph](https://github.com/joernio/flatgraph).

Below are benchmarks measuring various properties of how each database performs.

### Method

Our evaluation will consist of gathering metrics from performing graph operations commonly related to processing program intermediate graph representations. For precise benchmarking, we use the Java Microbenchmark Harness (JMH) [1] to measure each of the tasks in our benchmark. Four property graph databases are used in the evaluation: Apache TinkerGraph, Neo4j (embedded mode), OverflowDB, and flatgraph. All databases run as Java processes, so we are able to compare each database as closely as possible.

The benchmark aims to evaluate the following questions:

* **RQ1**: How does each databases' write performance scale when constructing an AST?
* **RQ2**: How does each databases' read performance scale while traversing the graph?
* **RQ3**: What is the memory footprint of each database at the end of each experiment?
* **RQ4**: What is the impact on storage when each database serializes the in-memory data to disk?

The benchmark uses programs from the Defects4j dataset [2] as it comprises a selection of real-world open-source Java applications and libraries of varying sizes and complexity. For the purpose of this benchmark, we retain the same identifiers for each JAR, and take the latest release of each JAR at the time of writing. Details for each program and version is found in Table 1.

**Table 1**: The programs of Defects4j, showing version used, file size, and AST order (`|V|`). The list is sorted by in ascending order with respect to the size of the AST order of the resulting graph.

| Name            | Version  | File Size (KB) | `\|V\|`   |
| --------------- | -------- | -------------- | --------- |
| Cli             | 1.8.0    | 71.87          | 18,177    |
| Csv             | 1.11.0   | 54.86          | 18,341    |
| JacksonXml      | 2.17.2   | 126.15         | 41,009    |
| Gson            | 2.11.0   | 291.44         | 67,776    |
| Codec           | 1.17.0   | 363.88         | 103,583   |
| Jsoup           | 1.18.1   | 442.33         | 116,664   |
| JxPath          | 1.3      | 292.96         | 121,135   |
| Mockito         | 5.12.0   | 687.48         | 121,496   |
| Collections     | 4.4      | 734.29         | 173,385   |
| Time            | 2.12.7   | 623.47         | 205,050   |
| Lang            | 3.14.0   | 642.53         | 214,912   |
| JacksonCore     | 2.17.2   | 568.29         | 271,937   |
| Compress        | 1.26.2   | 1,058.24       | 366,456   |
| JacksonDatabind | 2.17.2   | 1,610.79       | 457,200   |
| Chart           | 1.5.5    | 1,612.40       | 598,841   |
| Math            | 3.6.1    | 2,161.68       | 924,841   |
| Closure         | 20240317 | 13,657.18      | 3,688,170 |


Joern's JVM bytecode frontend (`jimple2cpg`) is used to construct each AST from the application JAR of each program, in other words, dependencies are excluded. During AST construction, the benchmark framework intercepts incoming graph transactions from the language frontend and applies them as a bulk transaction to the selected backend graph database. This allows us to isolate benchmark performance differences to the backend database.

**RQ1** concerns database write speed, and the average time to create an AST is measured over 10 iterations, with 1 warm up iteration, and a time-out of 6 minutes. **RQ2** considers query response times, or read speed, and thus the average time across several graph queries for each AST are measured over 3 iterations, with 1 warm up iteration, and a time-out of 5 minutes. Initial benchmarks show that read operations have lower variance than the write operations, thus the discrepancy in iterations per benchmark. The maximum heap size for both benchmarks is 8 GB.

The graph queries used under **RQ2**'s benchmark are the following:
  * _astDFS_: From every AST root, perform a depth first search until the leaves of the AST.
  * _astUp_: From every node, traverse up the AST until a root is encountered.
  * _orderSum_: For every node with the `ORDER` property, sum these values together.
  * _callOrderTrav_: Count the number of nodes with label `CALL` and `ORDER` with a value greater than 2 in an aggregate-style query.
  * _callOrderExplicit_: Same operation as _callOrderTrav_, except done via explicitly referencing each node and property directly for the comparison.
  * _indexedMethodFullName_: From a randomly gathered list of method full names, search for matching `METHOD` nodes, using indexes if supported.
  * _unindexedMethodFullName_: Same operation as _indexedMethodFullName_, except by disabling the use of indexing.

Throughout the aforementioned benchmarks, the JVM heap profiler developed by cache2k [3] is attached to JMH to report the heap usage for each experiment.

#### Results

All experiments were performed on a platform with an 6-core x86 CPU (3.4 GHz) and 32 GB memory. Java 21.0.2 was used with the _Z_ garbage collector (`-XX:+UseZGC`). Benchmarks can be reproduced at https://github.com/plume-oss/plume. Projects are ordered similarly to Table 1, and an error indicates either a timeout or out-of-memory error. 

Figure 1 addresses **RQ1** by presenting the results of AST creation performance across each database.  Among the general-purpose databases, Neo4j exhibits the poorest performance, timing out starting from the project "Time", with the exception of "JacksonCore". TinkerGraph consistently underperforms compared to both OverflowDB and flatgraph across most projects. A notable pattern emerges when comparing OverflowDB and flatgraph: starting from the project "Compress", OverflowDB experiences significant performance degradation, with its execution times displaying considerable variance. This behavior is likely attributable to the overflow mechanism. In contrast, flatgraph demonstrates more stable scaling performance under similar memory constraints, although it incurs higher write-speed overhead, likely due to its copy-on-write mechanism.

{{< figure src="/benchmarks/createAst.pdf" caption="**Figure 1**: The performance of AST creation for each database over each project at a fixed max heap memory of 8GB.">}}

Next, **RQ2** is addressed in Figure 2, which presents the read speeds for the graph queries outlined in the previous section. The error bars for Neo4j here are brought forward from the timeouts during AST creation, while for TinkerGraph, at "Closure" there was an out-of-memory error. Notably, flatgraph significantly outperforms the other databases, consistently demonstrating read speeds that are orders of magnitude faster. This ranges from about 50% in the worst case for _callOrderTrav_, and well over 99.5% in _astDFS_ and _astUp_ in the best case.

{{< figure src="/benchmarks/querySpeeds.pdf" caption="**Figure 2**: The performance of average query response times for the seven AST traversal benchmarks.">}}

In the multi-hop AST traversal queries, specifically _astDFS_ and _astUp_, TinkerGraph and Neo4j show better performance than OverflowDB until "JacksonDatabind". This may be a result of memory limitations or indicate scalability issues. For the three queries involving property accesses under varying conditions -- _orderSum_, _callOrderTrav_, and _callOrderExplicit_ -- a similar trend is observed. However, OverflowDB surpasses the other databases earlier, around the dataset "Time", except in the case of _callOrderTrav_. This could suggest that TinkerGraph and Neo4j gain an advantage from query planning, though Neo4j's scalability limitations become evident in these experiments.

Lastly, in the _MethodFullName_ string equality queries, OverflowDB outperforms both Neo4j and TinkerGraph. As TinkerGraph lacks indexing capabilities, its performance in both the indexed and unindexed variants remains comparable.

Figure 3 presents the average memory consumption, as reported by `jmap`, at the conclusion of each read-query experiment from Figure 2. Neo4j exhibits the highest memory usage by a significant margin, while OverflowDB and TinkerGraph show comparable performance. In contrast, flatgraph demonstrates a notable improvement in memory efficiency, consistently reducing memory consumption by 25-30% compared to OverflowDB.

{{< figure src="/benchmarks/memory.pdf" caption="**Figure 3**: The memory used by live objects as reported by `jmap` for each database over each project at a fixed max heap memory of 8GB.">}}

Figure 4 demonstrates the storage footprint left by each database after data serialization. flatgraph, with its naturally compressed encoding approach, consistently occupies only a small fraction of the storage space compared to the other databases. Neo4j and TinkerGraph exhibit lower storage usage than OverflowDB for smaller programs, but OverflowDB scales more efficiently for larger datasets.

{{< figure src="/benchmarks/storage.pdf" caption="**Figure 4**: The storage used by serialized AST data from each database.">}}

To further investigate the scalability differences under extreme conditions, we benchmark the generation of a complete code property graph for Linux 4.1.16 using Joern's C/C++ frontend, and evaluated both OverflowDB and flatgraph. Increased hardware resources were necessary, leading to using a platform featuring an 8-core x86 CPU (3.6 GHz) and 128 GB of memory. Java 17.0.12 was used with a max heap size value of 90 GB. The resulting graph contains 48 million nodes with 630 million properties (predominantly `String` and `Integer`), as well as 431 million edges with 115 million properties (all `String`). The results, presented in Table 2, show approximately 40% lower memory usage, 60% less total memory required, faster traversal times, and an 85% smaller file size.

**Table 2**: The metrics gathered from a one-shot CPG generation of Linux 4.1.16 using Joern (via `importCpg`) comparing OverflowDB and flatgraph.

|                           | OverflowDB  | flatgraph |
| ------------------------- | ----------- | --------- |
| Joern Version             | v2.0.448    | v4.0.1    |
| Final Heap Size (post-GC) | 32.90 GB    | 19.95 GB  |
| Time                      | 18.18 min   | 11.97 min |
| File Size                 | 2,554.64 MB | 624.93 MB |

#### Related Work

In more recent work, unrelated to the CPG, [4] developed an approach for storing Java source code in Neo4j, offering both remote and embedded query capabilities. Their system, ProgQuery, was evaluated against Frappe [5] and Semmle's CodeQL [6]. Similarly, [7] and [8] explored the analysis of PHP programs using Neo4j, though their work primarily focused on integrating Neo4j as a backend without a direct comparison to other methods.

Recent advancements inspired by the CPG [9] have targeted various languages and platforms. Graft [10] employs Apache TinkerPop for graph storage, while JAW [11] and the system by [12] use Neo4j to analyze JVM bytecode, JavaScript, and PHP, respectively.

Unrelated to property graphs, logic programming approaches, such as using Datalog [13, 14], as a domain-specific language, are frequently used to concisely build static analyzers that may be easier to reason about. While Datalog enables the construction of declarative queries that succinctly express an analysis, this often comes at the cost of reduced performance. [15] address this scalability challenge with Souffle, a highly optimized Datalog engine designed for large-scale static analysis tasks. The Doop framework has improved its runtime performance by adopting Souffle [16]. The Joern project, in contrast, has opted to remain backed by property graph databases, as they offer a lower barrier to entry, allowing for imperative interaction with queried data, integration with visualization tools, and more flexibility in schema and data specification. As discussed in this paper, general-purpose property graph databases are similarly unsuitable for static analysis; however, we suggest that flatgraph offers a comparable improvement to that which Souffl'e brought to Datalog-based frameworks. 


#### References

[1] OpenJDK Community, "Java microbenchmark harness (jmh)," https://openjdk.org/projects/code-tools/jmh/, 2024, version 1.37. [Online].  
[2] R. Just, D. Jalali, and M. D. Ernst, "Defects4j: A database of existing faults to enable controlled testing studies for java programs," in _Proceedings of the 2014 international symposium on software testing and analysis_, 2014, pp. 437–440.  
[3] J. Wilke, "cache2k java caching: Benchmarks," https://cache2k.org/benchmarks.html, 2016, accessed: August 2024.  
[4] O. Rodriguez-Prieto, A. Mycroft, and F. Ortin, "An efficient and scalable platform for java source code analysis using overlaid graph representations," _IEEE Access_, vol. 8, 2020.  
[5] N. Hawes, B. Barham, and C. Cifuentes, "Frappe: Querying the linux kernel dependency graph," in _Proc. of the GRADES'15_, 2015.  
[6] Semmle, "Codeql," https://semmle.com/codeql, 2024, retrieved July 2023.  
[7] D. Sadyrin, A. Dergachev, I. Loginov, I. Korenkov, and A. Ilina, "Application of graph databases for static code analysis of web-applications." in _MICSECS_, 2019.  
[8] J. Liu, "Enabling static program analysis using a graph database," Master’s thesis, Wright State University, 2020.  
[9] F. Yamaguchi, N. Golde, D. Arp, and K. Rieck, "Modeling and discovering vulnerabilities with code property graphs," in _Proc. of IEEE Symposium on Security and Privacy_, 2014.  
[10] W. Keirsgieter and W. Visser, "Graft: Static analysis of java bytecode with graph databases," in _Conf. of the South African Institute of Computer Scientists and Information Technologists_, 2020.  
[11] S. Khodayari and G. Pellegrino, "Jaw: Studying client-side csrf with hybrid property graphs and declarative traversals," in _Proc. of USENIX Security Symposium_, Vancouver, B.C., 2021.  
[12] M. Backes, K. Rieck, M. Skoruppa, B. Stock, and F. Yamaguchi, "Efficient and flexible discovery of PHP application vulnerabilities," in _IEEE European Symposium on Security and Privacy (EuroS&P)_, 2017.  
[13] N. Allen, B. Scholz, and P. Krishnan, "Staged points-to analysis for large code bases," in _Compiler Construction_, 2015, pp. 131–150.  
[14] N. Allen, P. Krishnan, and B. Scholz, "Combining type-analysis with points-to analysis for analyzing java library source-code," in _Proc. of the 4th ACM SIGPLAN International Workshop on State Of the Art in Program Analysis_, ser. SOAP 2015. New York, NY, USA: Association for Computing Machinery, 2015, p. 13–18.  
[15] H. Jordan, B. Scholz, and P. Subotic, "Souffle: On synthesis of program analyzers," in _Computer Aided Verification: 28th International Conference, CAV 2016, Toronto, ON, Canada, July 17-23, 2016, Proceedings, Part II 28_. Springer, 2016, pp. 422–430.  
[16] T. Antoniadis, K. Triantafyllou, and Y. Smaragdakis, "Porting DOOP to Souffle: a tale of inter-engine portability for datalog-based analyses," in _Proceedings of the 6th ACM SIGPLAN International Workshop on State Of the Art in Program Analysis_, 2017, pp. 25–30.  