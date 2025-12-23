[[ML Schema]]
## Language agnostic operators
**RunML**: 
	$RunML(i, ml)$ passes to a set of one or more Machine Learning models $ml$ a set $i$ of one or more input values, and returns the outputs of the models in $ml$. 
**GetML**
	$GetML(rt)$  deduces a set of one or more Machine Learning models from a set $rt$ of one or more runtimes. 
## ML Clause Grammar
```ebnf
MLClause = "RUN" input "ON" runtime "TO" output
```
## Semantics
The ML Clause is the equivalent of executing $RunML(i, GetML(rt))$, with the input set $i$ and runtime set $rt$. Below, the keywords used in the ML Clause are explained
- ```RUN```
	- Its argument ```input```is either a variable or a constant value
	- ```input``` represents the inputs to be passed to the Machine Learning model(s) defined in the Knowledge Graph
	- `input` is the equivalent of the input value set $i$ of $RunML$.
		- If `input`is a constant value, then $i$ is a set with a single element, which is that constant value
		- Otherwise $i$ is a set with multiple elements
- ```ON``` 
	- Its argument ```runtime```is either a variable or a constant value
	- `runtime` represents a runtime as defined in ML Schema
		- Each runtime is associated with a single Machine Learning model
			- `runtime` is used to obtain these ML models
		- If `runtime` is a variable, then many ML models are retrieved
	- `runtime` can be seen as the equivalent of the $rt$ argument of the $GetML$ function
		- If `runtime` is a constant value, then $rt$ contains only a single value, which means that $GetML$ returns only a single Machine Learning model
		- Otherwise, $rt$ contains as many elements as in `runtime`, and $GetML$ returns as many ML models as elements in $rt$, which has the same number of elements as in `runtime`
- ```TO```
	- Its argument ```output``` is a variable
	- ```output``` represents the variable in which the outputs generated from the obtained Machine Learning models are stored
	- It can be seen as the variable where the output of $RunML(i, GetML(rt))$ is stored

##  High-level overview of how $GetML(rt)$ can be implemented

![[MLSchemaNotation.png]]
- For each runtime  in $rt$, each represented by a node of ```rdf:type``` ```mls:Run```
	- For each ```mls:hasOutput``` relation starting at the ```mls:Run``` node
		- If one of these relations lead to a node of ```rdf:type``` ```mls:Model```, then that node contains the ML model corresponding to the runtime
- The retrieval of runtimes and their associated Machine Learning models can be represented by the following SPARQL query
```sparql 
SELECT ?ml, ?r
WHERE{
	?r rdf:type mls:Run.
	?r mls:hasOutput ?ml.
	?ml rdf:type mls:Model.
}
``` 

## Integration of the extension into the query plan
### Query plan representation
```rust
pub enum MLElem {
	Var {
		var: LogicalOperator
	}
	Const {
		cst: Term
	}
}
```
```rust 
LogicalOperator::MLRun(input: MLElem, mlmodels: LogicalOperator, output: Vec<String>)
```
- `input`
	- If it is a variable, so of type `MLElem::Var`, then it represents the query plan representation of the logical operator used to retrieve the `input` argument ML Clause
		- In this case, the logical operator `var` that is wrapped by `MLElem::Var` is of type `LogicalOperator`
	- Otherwise, if it is specified to be a constant, so of type `MLElem::Const`, then it represents the actual input value to be passed to one or more Machine Learning models
		- We can think of it in this case as the argument $i$ of the $RunML$ operator which only contains a single value, `input`
- `mlmodels`
	- Query plan representation of the logical operator used to retrieve Machine Learning models
	- If the argument of the `ON` keyword was a variable, then the logical operator of `mlmodels` also retrieves the runtimes from which these Machine Learning models were deduced
- `output`
	- String representation of the variable which is the argument of the `TO` keyword
	- This variable will store the outputs of the Machine Learning models defined in `mlmodels`

### Query plan building process
#### Explanation of the query plan building algorithm
In the initial query building process, for each pattern in the `WHERE` clause, a `LogicalOperator::Scan` logical operator is created for each of them. All these operators are then stored in a vector called `scan_operators`. Before adding these Scan operators to the vector, if there are filters that can be pushed down onto these operators, `LogicalOperator::Filter` logical operators are applied to these Scan operators. The operators in this vector are then ordered in ascending order based on their cost. Thus, the least costly operator is at the beginning of `scan_operators` while the costliest operator is at the end of this vector. Then, we initialise a variable called `result` which takes as value the first element of `scan_operators`.  For each operator `op` in `scan_operators`, excluding the first one, a `LogicalOperator::Join` logical operator with as first argument `result` and as second argument `op` is assigned to `result`.

To extend SPARQL with the new clause, this last step of joining the elements of `scan_operators` needs to be extended. In this situation, in case the `input` argument passed to the `RUN` keyword of the `RUN ... ON ... TO ...` clause is a variable, the last element of `scan_operators`, which contains a Scan operator that scans a pattern containing this variable, is first determined. Let us call this last element `lastOp` and the variable `inputVar`. A `LogicalOperator::Join` operator with as first argument `result` and as second argument `lastOp`. A `LogicalOperator::Projection` logical operator is then applied to this Join to project it on the `inputVar` variable. This Projection operator corresponds to extracting the values of the `inputVar` variable from the result of the execution of the Join operator. The Projection operator is also passed to the `LogicalOperator::MLRun` logical operator as its first argument. This first argument can be seen as the logical operator that is used to retrieve the argument $i$ of $RunML$, which represent the inputs to be passed to the single or multiple Machine Learning models.

Meanwhile, we chain a series of logical operators that should produce the result of the query below when executed, if the argument passed to the `ON` keyword of ML Clause is defined to be a variable named `?r`.
```sparql 
SELECT ?ml, ?r
WHERE{
	?r rdf:type mls:Run.
	?r mls:hasOutput ?ml.
	?ml rdf:type mls:Model.
} 
``` 
This chain of logical operators involves applying a Projection on the `?ml` and `?r` variables of a Join which has as its first parameter another Join and as its second parameter the Scan on the `?r mls:hasOutput ?ml.` pattern. This inner Join has as its first parameter a Scan on the `?r rdf:type mls:Run.` pattern, and as its second parameter a Scan on the `?ml rdf:type mls:Model.` pattern . This Projection, which retrieves all the values of the `?ml`, `?r` variables when executed, is passed as the second argument of the `LogicalOperator::MLRun` logical operator. The `?ml` variable correspond to the $ml$ argument $GetML$, which represents all the Machine Learning models to which the inputs $i$ need to be passed to. Meanwhile, `?r` represents the set of runtimes from which the Machine Learning models stored in `?ml` are deduced. However, if the argument passed to the `ON` keyword of ML Clause is defined to be a constant, then instead of the Projection on the `?ml` and `?r` variables, we just project on the `?ml` variable. Thus, the query that will be executed in this case with the chain of logical opertors will be as follows:
```sparql 
SELECT ?ml
WHERE{
	someRTCstName rdf:type mls:Run.
	someRTCstName mls:hasOutput ?ml.
	?ml rdf:type mls:Model.
} 
``` 

The third argument of this logical operator is a vector with a single string which indicates the variable where the outputs of the single or multiple Machine Learning models should be stored. We then set the value of `result` to that of a Join that has as its first argument `result` and as its second argument this `LogicalOperator::MLRun` logical operator. For the remaining elements in the `scan_operators` vector, we can join them together with the Join operator in the same way as described in the last sentence of the first paragraph.
#### Pseudocode of the query plan building algorithm
1. `mlparams = (input_str: &str, run_str: &str, output_str: &str)` represents the parsed ML operator
2. `patterns` is the set of triples, defined in the `WHERE` clause, that indicate the information that should be retrieved from the Knowledge Graph
3. If `input_str` is a variable (starts with ?)
	1. Initialise vector `input_var` with `input_str` in it
	2. Associate `input_str` with `input` of type `Term::Variable`
4. If `run_str` is a variable (starts with ?)
	1. Initialise vector `run_var` with `run_str` in it
	2. Associate `run_str` with `runs` of type `Term::Variable`
5. Put `output_str` in vector `output`
6. Let `scan_operators` be a vector that will store all the `LogicalOperator::Scan` logical operators
7. For each `(subject_str: &str, predicate_str: &str, object_str: &str)` in `patterns`
	1. If `subject_str` is a variable (starts with ?), associate it with `subject` of type `Term::Variable`
	2. Otherwise, associate `subject_str` with `subject` of type `Term::Constant`
	3. If `predicate_str` is a variable (starts with ?), associate it with `predicate` of type `Term::Variable`
	4. Otherwise, associate `predicate_str` with `predicate` of type `Term::Constant`
	5. If `object_str` is a variable (starts with ?), associate it with `object` of type `Term::Variable`
	6. Otherwise, associate `object_str` with `object` of type `Term::Constant`
	7. Create a Scan operator `scan_op = LogicalOperator::Scan((subject, predicate, object))`
	8. `filter_op = scan_op`
	9. For each filter that can be pushed down to a Scan (it is a `FilterExpression::Comparison` and the variables they compare are present in the scans)
		1. Let `condition` be the condition in the filter
		2. `filter_op = LogicalOperator::Selection(filter_op, condition)`
	10. Add `filter_op` to the `scan_operators` vector
8. Sort operators in `scan_operators` by cost
9. Let `sawInput` be false
10. Let `result` be the first element in `scan_operators`
11. If a variable in `result` corresponds to the one of the `input_str` element of `mlparams`
	1. Set `sawInput` to true
12. For each `op` in `scan_operators`, excluding the first element
	1. If `input_str` was a variable (`input_var` is not empty)
		1. If `op` is a  `LogicalOperator::Selection`
			1. Let `la` be the first (leftmost) argument of `op`
			2. Until `la` is of type `LogicalOperator::Scan`
				1. Set the value of `la` to the first argument of `la`
			3. If a variable in `la` corresponds to the `input_str` element of `mlparams`
				1. Set `sawInput` to true
			4. Otherwise (`la` has no variable corresponding to `input_str`)
				1. If `sawInput` is true
					1. Get the query plan representation of getting all the nodes of `rdf:type` `mls:Model` as `runselect`: 
						1. if `run_str` is a variable
							1. Let `run` be a `Term::Variable` associated to `run_str`
						2. Otherwise (`run_str` is not a variable)
							1. Let `run` be a `Term::Constant` associated to `run_str`
						3. Let `ml` be a `Term:Variable`
						4. Let `mlvec` be the vector which has as elements:
							1. the name of the `ml` `Term::Variable` as a String
							2. If `run_str` is a variable:
								1. `run_str` as a String
						5. Let `mt` be a `Term::Constant` associated with the `mls:Model` value
						6. Let `type` be a `Term::Constant` associated with the `rdf:type` value
						7. Let `ho` be the `Term:Constant` associated with the `mls:hasOutput` value
						8. Let `rt` be a `Term::Constant` associated with the `mls:Run` value
						9. Add triples `(run, type, rt)`, `(ml, type, mt)` and `(run, ho, ml)` to a vector called `sortMlOps`
						10. Sort the elements in `sortMLOps` by cost
						11. Initialise `runselect` with as value the first element of `sortMLOps` 
						12. For each `mlop` in `sortMLOps`, excluding its first element
							1. `runselect = LogicalOperator::Join(runselect, mlop)`
						13. `runselect = LogicalOperator::Projection(runselect, mlvec)`
					2. `result = LogicalOperator::Join(result, LogicalOperator::MLRun(MLElem::Var(LogicalOperator::Projection(result, input_var)), MLElem::Var(runselect), output))`
		2. Otherwise (`op` is a `LogicalOperator::Scan`)
			1. If the variable in `op` corresponds to the one of the `input_str` element of `mlparams`
				1. Set `sawInput` to true
			2. Otherwise (`la` has no variable corresponding to `input_str`)
				1. If `sawInput` is true
					1. Get the query plan representation of getting all the nodes of `mls:Model` as `runselect`: 
						1. If `run_str` is a variable
							1. Let `run` be a `Term::Variable` associated to `run_str`
						2. Otherwise (`run_str` is not a variable)
							1. Let `run` be a `Term::Constant` associated to `run_str`
						3. Let `ml` be a `Term:Variable`
						4. Let `mlvec` be the vector which has as elements:
							1. the name of the `ml` `Term::Variable` as a String
							2. If `run_str` is a variable:
								1. `run_str` as a String
						5. Let `type` be a `Term::Constant` associated with the `rdf:type` value
						6. Let `mt` be a `Term::Constant` associated with the `mls:Model` value
						7. Let `type` be a `Term::Constant` associated with the `rdf:type` value
						8. Let `ho` be the `Term:Constant` associated with the `mls:hasOutput` value
						9. Let `rt` be a `Term::Constant` associated with the `mls:Run` value
						10. Add triples `(run, type, rt)`, `(ml, type, mt)` and `(run, ho, ml)` to a vector called `sortMlOps`
						11. Sort the elements in `sortMLOps`
						12. Initialise `runselect` with as value the first element of `sortMLOps` 
						13. For each `mlop` in `sortMLOps`, excluding its first element
							1. `runselect = LogicalOperator::Join(runselect, mlop)`
						14. `runselect = LogicalOperator::Projection(runselect, mlvec)`
					2. `result = LogicalOperator::Join(result, LogicalOperator::MLRun(MLElem::Var(LogicalOperator::Projection(result, input_var)), MLElem::Var(runselect), output))`
					3. Set `sawInput` to `false`
	2. `result = LogicalOperator::Join(result, op)`
13. If `input_str` is a constant (`input_var` is empty)
	1. Let `input_const` be the `Term::Constant` associated with `input_str`
	2. Get the query plan representation of getting all the nodes of `mls:Model` as `runselect`: 
		1. If `run_str` is a variable (starts with ?)
			1. Let `run` be a `Term::Variable` associated to `run_str`
		2. Otherwise
			1. Let `run` be a `Term::Constant` associated to `run_str`
		3. Let `ml` be a `Term:Variable`
		4. Let `mlvec` be the vector which has as elements:
			1. the name of the `ml` `Term::Variable` as a String
			2. If `run_str` is a variable:
				1. `run_str` as a String
		5. Let `type` be a `Term::Constant` associated with the `rdf:type` value
		6. Let `mt` be a `Term::Constant` associated with the `mls:Model` value
		7. Let `ho` be the `Term:Constant` associated with the `mls:hasOutput` value
		8. Let `rt` be a `Term::Constant` associated with the `mls:Run` value
		9. Add triples `(run, type, rt)`, `(ml, type, mt)` and `(run, ho, ml)` to a vector called `sortMlOps`
		10. Sort the elements in `sortMLOps`
		11. Initialise `runselect` with as value the first element of `sortMLOps` 
		12. For each `mlop` in `sortMLOps`, excluding its first element
			1. `runselect = LogicalOperator::Join(runselect, mlop)`
		13. `runselect = LogicalOperator::Projection(runselect, mlvec)`
	3. `result = LogicalOperator::Join(result, LogicalOperator::MLRun(MLElem::Const(input_const), MLElem::Var(runselect), output))`
14. Return `result`

### Physical plan presentation
```rust
pub enum MLElemPhysical {
	Var {
		var: LogicalOperator
	}
	Const {
		cst: Term
	}
}
```
```rust 
PhysicalOperator::MLExecute(input: MLElemPhysical, mls: PhysicalOperator, output: Vec<String>)
```
- `input`: 
	- Physical plan representation of the input argument ML Clause
	- If it is a concrete value, it is of type `Term`
	- Otherwise, it is of type `LogicalOperator`
- `mls`: physical plan representation of the ML model(s), deduced from the runtime(s)
- `output`: `Term::Variable` representation of the variable which is the argument of the `TO` keyword

### Converting the logical operator to a physical operator
This conversion from the logical operator to a physical operator is done within the `find_best_plan_recursive` function. The algorithm described below indicates only the extra condition that should be added to the `match` operator in the `find_best_plan_recursive`, while not deleting any of the existing code in the function
#### Pseudocode of the match extension needed to process the ML Clause
1. `logical_plan` is a `LogicalOperator::MLRun(input: MLElem, mlmodels: LogicalOperator, output: Vec<String>)`
	1. If `logical_plan.input` is of type `MLElem::Const`
		1. Convert `logical_plan.input` to a value of type `MLElmPhysical::Const`, which is then assigned to a variable `inputPhysical`
	2. Otherwise (`logical_plan.input` is of type `MLElem::Var`)
		1. `inputPhysical = find_best_plan_recursive(logical_plan.input.var)`
		2. `inputPhysical = MLElemPhysical::Var(inputPhysical)`
	3. `mlphysical = find_best_plan_recursive(logical_plan.mlmodels)`
	4. Returns `PhysicalOperator::MLExecute(inputPhysical, mlphysical, logical_plan.output)`
#### Explanation of the match extension needed to process the ML Clause
If the `logical_plan` given as argument to `find_best_plan_recursive` is of type `LogicalOperator::MLRun`, we first check its `input` argument. If it is a constant value, which means that it is of type `MLElem::Const`, then we convert it to a value of type `MLElmPhysical::Const`, which we assign to a variable called `inputPhysical`.

Otherwise, `input` is of type `MLElem::Var`. In this case, we need to call `find_best_plan_recursive` on the `LogicalOperator` value stored in `input` to get a `PhysicalOperator` corresponding to it, which is then assigned to the `inputPhysical` variable. The resulting `PhysicalOperator` obtained after passing `input` to `find_best_plan_recursive` will have experienced optimisations such as an optimal reordering of the arguments of its Joins.

Then, with the `mlmodels` argument of `LogicalOperator::MLRun` being a `LogicalOperator` itself, we can pass it back to the `find_best_plan_recursive` function, which will return an optimised `PhysicalOperator`. The output of `LogicalOperator::MLRun` will be assigned to the `mlphysical` variable.

Finally, the physical representation of the `LogicalOperator::MLRun` logical operator of the ML Clause is `PhysicalOperator::MLExecute(inputPhysical, mlphysical, output)`, where the `output` argument is the same as the `output` argument of the `logical_plan`.
### Processing the physical operator
This physical operator execution is done within the `execute_with_ids` function. This function returns a Vector of HashMaps, which model a table. The HashMaps represent the rows of the table. For each element in the HashMap, the key corresponds to the column name (which corresponds to a variable) in which the value mapped to it belongs to. 

Just like in the previous section, the algorithm described below only describes the extra condition that should be added to the `match` operator of `execute_with_ids` to execute the physical operator of ML Clause.
#### Pseudocode of the match extension needed to execute the ML Clause
1. `operator` is a `PhysicalOperator::MLExecute(input: MLElemPhysical, mls: PhysicalOperator, output: Vec<String>)`
	1. If `operator.input` is of type `MElemPhysical::Var`
		1. `inputArg = execute_with_ids(operator.input.var)`
			1. `inputArg` corresponds to a table with a single column, representing the input set $i$ of $GetML$, or the `input` variable passed to the `RUN` keyword of the ML Clause
			2. This means that `inputArg` is a Vector with HashMaps that all have a single key-value pair in them, with the key corresponding to the `input` variable
		2. `modelsToRun = execute_with_ids(operator.mls)`
			1. `modelsToRun` is also a Vector with HashMaps that all have a either a single or two key-value pair in them
		3. Initialise a vector called `finalResults`
		4. For each `runRow`HashMap in `modelsToRun`
			1. If `runRow` has 2 elements
				1. The first element, which is a key-value pair, is assigned to `mlVal`
				2. The second element, which is also a key-value pair, is assigned to `runVal`
			2. Otherwise (`runRow` has only one element)
				1. The first element, which is a key-value pair, is assigned to `mlVal`
			3. For each `row` in `inputArg`
				1. Initialise a HashMap called `newRow`
				2. Add `row`, `mlVal`, and if it exists, `runVal` to `newRow`
				3. Pass to the value in `row` to the value in `mlVal`, and store its output as a value in a key-value pair `outputVal` with as key the only String stored in `operator.output`
				4. Add `outputVal` to `newRow`
				5. Add `newRow` to `finalResults`
	2. Otherwise (`operator.input` is of type `MElemPhysical::Const`)
		1. `modelsToRun = execute_with_ids(operator.mls)`
		2. For each `runRow`HashMap in `modelsToRun`
			1. If `runRow` has 2 elements
				1. The first element, which is a key-value pair, is assigned to `mlVal`
				2. The second element, which is also a key-value pair, is assigned to `runVal`
			2. Otherwise (`runRow` has only one element)
				1. The first element, which is a key-value pair, is assigned to `mlVal`
			3. Initialise a HashMap called `newRow`
			4. Add mlVal`, and if it exists, `runVal` to `newRow`
			5. Pass to the value in `row` to the value in `mlVal`, and store its output as a value in a key-value pair `outputVal` with as key the only String stored in `operator.output`
			6. Add `outputVal` to `newRow`


## Query execution example
### Query
```sparql
SELECT ?a ?b ?o ?d ?r ?e ?f ?g
WHERE{
	?f somerel ?g. #p1
	?a ?b ?c. #p2
	?a rel ?d. #p3
	RUN ?a ON ?r TO ?o.
	?o ?e ?d. #p4
}
```
### Logical plan creation phase
![[QueryParsingProcess.png]]


