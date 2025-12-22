[[ML Schema]]
## Notation
```sparql
RUN input ON runtime TO output
```
## Semantics
- ```RUN```
	- Its argument ```input```is either a variable or a constant value
	- ```input``` represents the inputs to be passed to the Machine Learning model(s) defined in the Knowledge Graph
- ```ON``` 
	- Its argument ```runtime```is either a variable or a constant value
	- ```runtime```represents a runtime as defined in ML Schema
		- Each runtime is associated with a single Machine Learning model
			- ```runtime``` is used to obtain these ML models
		- If ```runtime```is a variable, then many ML models are retrieved
- ```TO```
	- Its argument ```output``` is a variable
	- ```output``` represents the variable in which the outputs generated from the obtained Machine Learning models are stored

##  High-level overview of how to get Machine Learning models from runtimes

![[MLSchemaNotation.png]]
- For each runtime, represented by a node of ```rdf:type``` ```mls:Run```
	- For each ```mls:hasOutput``` relation starting at the ```mls:Run``` node
		- If one of these relations lead to a node of ```rdf:type``` ```mls:Model```, then that node contains the ML model corresponding to the runtime
- The retrieval of Machine Learning models can be represented by the SPARQL query
```sparql 
SELECT ?ml
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
- `input`: query plan representation of the input argument ML Clause
- `mlmodels`: query plan representation of the Machine Learning model(s), deduced from the runtimes passed as argument to the `ON` keyword
- `output`: string representation of the variable which is the argument of the `TO` keyword

### Query plan building process
The pseudocode below explains how the `build_logical_plan_optimized` function will now work
#### Pseudocode
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
			1. Let `la` left argument of `op`, which should be of type `LogicalOperator::Scan`
			2. If a variable in `la` corresponds to the one of the `input_str` element of `mlparams`
				1. Set `sawInput` to true
			3. Otherwise
				1. If `sawInput` is true
					1. Get the query plan representation of getting all the nodes of `rdf:type` `mls:Model` as `runselect`: 
						1. if `run_str` is a variable
							1. Let `run` be a `Term::Variable` associated to `run_str`
						2. Otherwise
							1. Let `run` be a `Term::Constant` associated to `run_str`
						3. Let `ml` be a `Term:Variable`
						4. Let `mlvec` be the vector which only has as element the name of the `ml` `Term::Variable` as a String
						5. Let `mt` be a `Term::Constant` associated with the `mls:Model` value
						6. Let `type` be a `Term::Constant` associated with the `rdf:type` value
						7. Let `ho` be the `Term:Constant` associated with the `mls:hasOutput` value
						8. Let `rt` be a `Term::Constant` associated with the `mls:Run` value
						9. Add triples `(run, type, rt)`, `(ml, type, mt)` and `(run, ho, ml)` to a vector called `sortMlOps`
						10. Sort the elements in `sortMLOps`
						11. Initialise `runselect` with as value the first element of `sortMLOps` 
						12. For each `mlop` in `sortMLOps`, excluding its first element
							1. `runselect = LogicalOperator::Join(runselect, mlop)`
						13. `runselect = LogicalOperator::Projection(runselect, mlvec)`
					2. `result = LogicalOperator::Join(result, LogicalOperator::MLRun(MLElem::Var(LogicalOperator::Projection(result, input_var)), MLElem::Var(runselect), output))`
		2. Otherwise
			1. If the variable in `op` corresponds to the one of the `input_str` element of `mlparams`
				1. Set `sawInput` to true
			2. Otherwise
				1. If `sawInput` is true
					1. Get the query plan representation of getting all the nodes of `mls:Model` as `runselect`: 
						1. If `run_str` is a variable
							1. Let `run` be a `Term::Variable` associated to `run_str`
						2. Otherwise
							1. Let `run` be a `Term::Constant` associated to `run_str`
						3. Let `ml` be a `Term:Variable`
						4. Let `mlvec` be the vector which only has as element the name of the `ml` `Term::Variable` as a String
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
13. If `input_str` was a constant (`input_var` is empty)
	1. Let `input_const` be the `Term::Constant` associated with `input_str`
	2. Get the query plan representation of getting all the nodes of `mls:Model` as `runselect`: 
		1. If `run_str` is a variable (starts with ?)
			1. Let `run` be a `Term::Variable` associated to `run_str`
		2. Otherwise
			1. Let `run` be a `Term::Constant` associated to `run_str`
		3. Let `ml` be a `Term:Variable`
		4. Let `mlvec` be the vector which only has as element the name of the `ml` `Term::Variable` as a String
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
#### Explanation of the pseudocode
In the initial query building process, for each pattern in the `WHERE` clause, a `LogicalOperator::Scan` logical operator is created for each of them. All these operators are then stored in a vector called `scan_operators`. The operators in this vector are then ordered in ascending order based on their cost. Thus, the least costly operator is at the beginning of `scan_operators` while the costliest operator is at the end of this vector. Then, we initialise a variable called `result` which takes as value the first element of `scan_operators`.  For each operator 

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
- `input`: physical plan representation of the input argument ML Clause
- `mls`: physical plan representation of the ML model(s), deduced from the runtime(s)
- `output`: `Term::Variable` representation of the variable which is the argument of the `TO` keyword

### Converting the logical operator to a physical operator
This conversion from the logical operator to a physical operator is done within the `find_best_plan_recursive` function
1. The type of the `input` parameter of `LogicalOperator::MLRun` corresponds to
	1. `MLElem::Var`
		1. `pinput = find_best_plan_recursive(input)`
	2. `MLElem::Const`
		1. Convert `input` to `MLElemPhysical` as `pinput`
2. `pmls = find_best_plan_recursive(LogicalOperator::MLRun.mlmodels)`
3. `poutput = LogicalOperator::MLRun.output`
4. Return `PhysicalOperator::MLExecute(pinput, pmls, poutput)`
### Processing the physical operator
This physical operator execution is done within the `execute_with_ids` function
1. `mliputs = execute_with_ids(PhysicalOperator::MLExecute.input)
2. `models = execute_with_ids(PhysicalOperator::MLExecute.mls)`
3. For `inputVal` in `mlinputs`
	1. For `model` in `models`
		1. Let `valml` be the value stored in the `model` element
		2. Let `val` be the value stored in `inputVal`
		3. `resultVal = valml(val)`
		4. Let `outputVal` be an element with as key the only element in `output` and as value `resultVal`
		5. Initialise a HashMap `resultRow` and add to it `inputVal`, `model` and `outputVal`
		6. Add `resultRow` to a vector `resultTable`
4. Return `resultTable`
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


