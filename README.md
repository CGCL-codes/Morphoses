
**Morphoses: A General Graph Stream Fine-grained Tracking System at Scale**


## 1. What is it?

GraphBolt is an efficient streaming graph processing system that provides Bulk Synchronous Parallel (BSP) guarantees. GraphBolt performs dependency-driven incremental processing which quickly reacts to graph changes, and provides low latency & high throughput processing. [[Read more]](https://www.cs.sfu.ca/~keval/contents/papers/graphbolt-eurosys19.pdf)

GraphBolt, now incorporates the DZiG run-time inorder to perform sparsity-aware incremental processing, thereby pushing the boundary of dependency-driven processing of streaming graphs. [[Read more]](https://www.cs.sfu.ca/~keval/contents/papers/dzig-eurosys21.pdf)

For asynchronous algorithms, GraphBolt incorporates KickStarter's light-weight dependency tracking and trimming strategy. [[Read more]](https://www.cs.sfu.ca/~keval/contents/papers/kickstarter-asplos17.pdf)

##  2. Getting Started

### 2.1 Core Organization

The `core/graphBolt/` folder contains the [GraphBolt Engine](#3-graphbolt-engine), the [KickStarter Engine](#4-kickstarter-engine), and our [Stream Ingestor](#5-stream-ingestor) module. The application/benchmark codes (e.g., PageRank, SSSP, etc.) can be found in the `apps/` directory. Useful helper files for generating the stream of changes (`tools/generators/streamGenerator.C`), creating the graph inputs in the correct format (`tools/converters/SNAPtoAdjConverter.C` - from Ligra's codebase), and comparing the output of the algorithms (`tools/output_comparators/`) are also provided.

### 2.2 Requirements
- g++ >= 5.3.0 with support for Cilk Plus.
- [Mimalloc](https://github.com/microsoft/mimalloc) - A fast general purpose memory allocator from Microsoft (version >= 1.6).
    - Use the helper script `install_mimalloc.sh` to install mimalloc.
    - Update the LD_PRELOAD enviroment variable as specified by install_mimalloc.sh script.

**Important: GraphBolt requires mimalloc to function correctly and efficiently.**


### 2.3 Compiling and Running the Application

Compilation is done from within `apps` directory. To compile, run
```bash
$   cd apps
$   make -j
```
 The executable takes the following command-line parameters:
 - `-s` : Optional parameter to indicate a symmetric (undirected) graph is used. 
 - `-streamPath` : Path to the input stream file or pipe (More information on the input format can be found in [Section 2.4](#24-graph-input-and-stream-input-format)).
 - `-numberOfUpdateBatches` : Optional parameter to specify the number of edge updates to be made. Default is 1.
 - `-nEdges` : Number of edge operations to be processed in a given update batch.
 - `-outputFile` : Optional parameter to print the output of a given algorithms.
 - Input graph file path (More information on the input format can be found in [Section 2.4](#24-graph-input-and-stream-input-format)).

For example,
```bash
$   # Ensure that LD_PRELOAD is set as specified by the install_mimalloc.sh
$   ./PageRank -numberOfUpdateBatches 2 -nEdges 1000 -streamPath ../inputs/sample_edge_operations.txt -outputFile /tmp/output/pr_output ../inputs/sample_graph.adj
$   ./LabelPropagation -numberOfUpdateBatches 3 -nEdges 2000 -streamPath ../inputs/sample_edge_operations.txt -seedsFile ../inputs/sample_seeds_file -outputFile /tmp/output/lp_output ../inputs/sample_graph.adj
$   ./COEM -s -numberOfUpdateBatches 3 -nEdges 2000 -streamPath ../inputs/sample_edge_operations.txt -seedsFile ../inputs/sample_seeds_file -partitionsFile ../inputs/sample_partitions_file -outputFile /tmp/output/coem_output ../inputs/sample_graph.adj
$   ./CF -s -numberOfUpdateBatches 2 -nEdges 10000 -streamPath ../inputs/sample_edge_operations.txt -partitionsFile ../inputs/sample_partitions_file -outputFile /tmp/output/cf_output ../inputs/sample_graph.adj.un
$   ./SSSP -source 0 -numberOfUpdateBatches 1 -nEdges 500 -streamPath ../inputs/sample_edge_operations.txt -outputFile /tmp/output/sssp_output ../inputs/sample_graph.adj
$   ./BFS -source 0 -numberOfUpdateBatches 1 -nEdges 50000 -streamPath ../inputs/sample_edge_operations.txt -outputFile /tmp/output/bfs_output ../inputs/sample_graph.adj
```
Other additional parameters may be required depending on the algorithm. Refer to the `Compute()` function in the application code (`apps/PageRank.C`, `apps/SSSP.C` etc.) for the supported arguments. Additional configurations for the graph ingestor and the graph can be found in [Section 5](#5-stream-ingestor).

### 2.4 Graph Input and Stream Input Format



For weighted graphs, the input graph should be in the weighted adjacency graph format. It is similar to [adjacency graph format](http://www.cs.cmu.edu/~pbbs/benchmarks/graphIO.html) but with the edge weights following the edges.

For example, a sample graph in SNAP format (weighted edgelist) and weighted adjacency graph format is shown below.

*SNAP format (weighted edgelist):*
```txt
0 1 10
0 2 100
```
*Weighted Adjacency Graph format:*
```txt
WeightedAdjacencyGraph
3
2
0
2
2
1
2
10
100
```
You can use `tools/converter/SNAPtoAdjConverter.C` to convert the weighted edgelist to the weighted adjacency graph format as follows:
```bash
$   ./SNAPtoAdjConverter -w inputGraphWeighted.snap inputGraphWeighted.adj
$   # for undirected (symmetric) graphs, use the -s flag
$   ./SNAPtoAdjConverter -s -w inputGraphWeighted.snap inputGraphWeightedUndirected.adj 
```

Each entry in the streaming weighted input file should be of the format `[d/a] source destination edge_data`, where `d` indicates deletion and `a` indicates addition.

*Streaming weighted input file:*
```txt
d   0   1   10
a   1   2   20
```

To use a weighted graph, compile the program with `WEIGHTED=1` as shown below:
```bash
$   make WEIGHTED=1 SSSP
$   ./SSSP -source 0 -numberOfUpdateBatches 1 -nEdges 1000 -streamPath ../inputs/sample_edge_operations.pipe -outputFile /tmp/output/sssp_output ../inputs/sample_graph.adj.weighted
```

The edge weight datatype should be defined similar to `apps/SSSP_edgeData.h` by extending the `EdgeDataType` struct defined under `core/graph/edgeDataType.h`. The following functions determine how the edge weight from the input files are transformed and used by the system:
- `createEdgeData(const char *edgeDataString)` - creates the edge data from the character string provided in graph input or streaming input. For example in SSSP, the string "10" is converted to the integer 10 and stored as edge weight. 
- `setEdgeDataFromPtr(EdgeDataType *edgeData)` - performs a deep copy of the passed edge data.
- `del()` - to deallocate any allocated memory.

**Complex edge data**

Graphs with complex edge data are also supported provided that the complex edge data is represented as a single string without spaces. For example, if each edge has `{edge_id, distance, max_speed}` it can be represented in the weighted SNAP format as follows:
```txt
0 1 1_0.3_0.5
0 2 2_1.5_1.68
```

This graph with complex edge data can be converted into the weighted adjacency graph format with `tools/converter/SNAPtoAdjConverter.C` as shown below.
```bash
$   ./SNAPtoAdjConverter -w inputGraphWeighted.snap inputGraphWeighted.adj
$   # for undirected (symmetric) graphs, use the -s flag
$   ./SNAPtoAdjConverter -s -w inputGraphWeighted.snap inputGraphWeightedUndirected.adj 
```

The weighted adjacency graph can then be used in user programs by defining the corresponding `EdgeDataType` struct as discussed above. In the `createEdgeData(const char *edgeDataString)` function, the string can be parsed into the respective datatypes `{long, double, double}` for `{edge_id, distance, max_speed}`.

The initial input graph should be in the [adjacency graph format](http://www.cs.cmu.edu/~pbbs/benchmarks/graphIO.html). 
For example, the SNAP format (edgelist) and the adjacency graph format for a sample graph are shown below.

SNAP format:
```txt
0 1
0 2
2 0
2 1
```
 Adjacency Graph format:
```txt
AdjacencyGraph
3
4
0
2
2
1
2
0
1
```
You can use `tools/converters/SNAPtoAdjConverter` to convert an input graph in Edgelist format (SNAP format) to the adjacency graph format, as follows:
```bash
$   ./SNAPtoAdjConverter inputGraph.snap inputGraph.adj
$   # for undirected (symmetric) graphs, use the -s flag
$   ./SNAPtoAdjConverter -s inputGraph.snap inputGraphUndirected.adj 
```
The streaming input file should have the edge operation (addition/deletion) on a separate line. The edge operation should be of the format, `[d/a] source destination` where `d` indicates edge deletion and `a` indicates edge addition. Example streaming input file:
```bash
a 1 2
d 2 3
a 4 5
...
```

Edge operations can be streamed through a pipe using `tools/generators/streamGenerator.C`. It takes in the following command-line parameters:
- `-edgeOperationsFile` : Input file containing the edge operations in the format mentioned above.
- `-outputPipe` : Path of the output pipe where the edges are streamed to.

```bash
$   cd tools/generators
$   make streamGenerator
$   ./streamGenerator -edgeOperationsFile ../inputs/sample_edge_operations.txt -outputPipe ../inputs/sample_edge_operations.pipe
```
More details regarding the ingestor can be found in [Section 5](#5-stream-ingestor).
Information regarding weighted graphs can be found in [Section 6](#6-Weighted-Graphs).

## 3. GraphBolt Engine

The GraphBolt engine provides Bulk Synchronous Parallel (BSP) guarantees while incrementally processing streaming graphs.

### 3.1 Creating Applications using the GraphBolt Engine

A key design decision of the GraphBolt framework is to ensure that the application code remains oblivious to GraphBolt's internal subtleties while still providing fast performance.

So, the application code only needs to express its computation using the following functions. More details regarding these functions can be found in the inline comments of `GraphBoltEngine.h`.

#### AggregateValue and VertexValue initialization:
- initializeAggregationValue()
- initializeVertexValue()
- aggregationValueIdentity()
- vertexValueIdentity()

GraphBolt stores information for each vertex in the form of aggregation values. So, first, the user should identify the aggregation value and the vertex value for the algorithm. For example in PageRank, the vertex value is its pagerank (`PR`) and the aggregation value is the sum of `(PR[u]/out_Degree[u])` values from all its inNeighbors. 

#### Activate vertex / Compute vertex for a given iteration:
- forceActivateVertexForIteration()
- forceComputeVertexForIteration()
- shouldUseDelta()

In iterative graph algorithms, at a given iteration `i`, a set of vertices will push some value to their outNeighbors. These are the active vertices for that iteration. The outNeighbors which receive these values will then compute their updated values. The following functions are provided to force a vertex to be either active/compute at a given iteration. For example, in Label Propagation, all the vertices should compute their values at each iteration irrespective of whether they receive any new changes from their inNeighbors at that iteration (refer `apps/LabelPropagation.C`).

#### Add to or remove from aggregation:
- addToAggregation()
- addToAggregationAtomic()
- removeFromAggregation()
- removeFromAggregationAtomic()

These are the functions used to add a value to or remove some value from the aggregation value. For sum, it is simply adding and subtracting the values from the aggregation value passed. Note that `addToAggregationAtomic()` and `removeFromAggregationAtomic()` will be called by multiple threads on the same aggregation value. So, the update should be performed atomically using CAS.

#### Edge functions:
- sourceChangeInContribution()
- edgeFunction()
- edgeFunctionDelta()

The edge operation is split into 3 phases:
1. Determine the source contribution - The computations for a given vertex which are dependent only on the source values are performed here. For example, in PageRank, a vertex `u` adds the value `PR[u]/out_degree[u]` to the aggregation value of all its outNeighbors. Since this computation of `PR[u]/out_degree[u]` is common for processing all the outEdges of `u`, we can compute this value (contribution of the source vertex) only once and perform the addition for all outEdges.
2. Transform the contribution depending on the edge data - In this step, the source vertex contribution is transformed by the edge property. For example in weighted page rank, the contribution will be multiplied by the edge weight.
3. Aggregating the contribution to the aggregation value using `addToAggregationAtomic()`.

Note that these functions do not require CAS or locks. In the case of complex aggregations, an additional `edgeFunctionDelta()` has to be defined. Refer the `apps/GraphBoltEngine.h`, `apps/GraphBoltEngine_complex.h` for further details of these functions.

#### Vertex compute function and determine end of computation:
- computeFunction()
- notDelZero()

Given an aggregation value, `computeFunction()` computes the vertex value corresponding to this aggregation value.
In order to detemine the convergence condition, the `notDelZero()` is used to determine whether the value of vertex has significantly changed compared to its previous value. 
Both these functions do not require CAS or locks as they will be invoked in a vertex parallel manner.

#### Determine how an edge update affects the source / destination:
- hasSourceChangedByUpdate()
- hasDestinationChangedByUpdate()

These functions are used to define how an edge update affects the source and destination vertex, i.e., whether the vertex should be activated or its value recomputed (using `computeFuntion()`) in the first iteration. For example, in PageRank, if the out_degree of a vertex changes, then it will be active in the first iteration. While in COEM, if the sum of inWeights of a vertex changes, then its value should be computed in the first iteration.

#### Compute function
- compute()

This is the starting point of the application. The GraphBolt engine is initialized here with the required configurations and started.

In addition to these functions, the algorithm also needs to define an `Info` class which contains all the global variable/constants required for that application. It should implement the following functions:
- copy()
- processUpdates()
- cleanup()



## 4. Stream Ingestor

The stream ingestor FIFO is specified by `-streamPath`. Edge operations can be written to this FIFO. `-nEdges` specifies the maximum number of edge operations that can be passed to the GraphBolt engine in a single batch. The GraphBolt engine will continue to receive batches of edges from the stream ingestor until either the stream is closed (when there are no more writers to the FIFO) or when `-numberOfBatches` has been exceeded. If the writing end of the FIFO is not opened, the GraphBolt engine (which is the reading end) will block and wait until it is opened. 

There are a few optional flags that can affect the behaviour and determine the validity of the edge operations  passed to the command line parameter `-streamPath`:

- `-fixedBatchSize`: Optional flag to ensure that the batch size is strictly adhered to. If the FIFO does not contain enough edges, the ingestor will block until it has received enough edges specified by `-nEdges` or until the stream is closed. 
- `-enforceEdgeValidity`: Optional flag to ensure that all edge operations in the batch are valid. For example, an edge deletion operation is valid only if the edge to be deleted is present in the graph. In the case of a `simple graph` (explained below), an edge addition operation is valid only if that edge does not currently exist in the graph. Invalid edges are discarded and are not included while counting the number of edges in a batch.
- `-simple`: Optional flag used to ensure that the input graph remains a simple graph (ie. no duplicate edges). The input graph is checked to remove all duplicate edges. Duplicate edges are not allowed within a batch and edge additions are checked to ensure that the edge to be added does not yet exist within the graph.
- `-debug`: Optional flag to print the edges that were determined to be invalid.


## 5. Acknowledgements
Some utility functions from [Ligra](https://github.com/jshun/ligra) and [Problem Based Benchmark Suite](http://www.cs.cmu.edu/~pbbs/index.html) are used as part of this project. We are thankful to them for releasing their source code.


