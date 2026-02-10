[[ML Schema]]
## Language agnostic operators
**RunML**: 
	$RunML(i, ml)$ passes to a set of one or more Machine Learning models $ml$ a set $i$ of one or more input features, and returns the outputs of the models in $ml$. Each input feature in $i$ is itself a set of one or more values the feature can take
**GetML**
	$GetML(rt)$  deduces a set of one or more Machine Learning models from a set $rt$ of one or more runtimes. 
## ML Clause Grammar
```ebnf
MLClause = "RUN {" inputs "} ON " runtime " TO " output
inputs = feature ", " inputs | ""
```
This clause should be used only within the `WHERE` clause
## Semantics
The ML Clause is equivalent to executing $RunML(i, GetML(rt))$, with the input set $i$ and runtime set $rt$. Below, the keywords used in the ML Clause are explained
- ```RUN```
	- Its argument set `inputs`, which contains the values `feature`, can be thought as the argument $i$ of $RunML$.
	- `feature` can be a variable or a constant value
		- If the value of a specific `feature` needs to be hardcoded, then `feature` is a constant
			- Otherwise, it is a variable
		- If `feature` is a variable, then the values it can have are restricted by the patterns in the `WHERE` clause which also contain `feature`.
		- **The `WHERE` clause NEEDS to have AT LEAST ONE pattern that also contains `feature` in case `feature` is a variable**
			- Otherwise, `feature` will take an infinite number of possible values.
	- **Define the `feature` values in the order they will be passed on to the ML model**
- ```ON``` 
	- Its argument `runtime` represents a runtime as defined in ML Schema
	- `runtime` contains at least one value
	- `runtime` corresponds to the $rt$ argument of the $GetML$ function
		- If `runtime` contains a single value, then executing $GetML$ with as argument `runtime` will return a set with only a single Machine Learning model
		- Otherwise, $GetML$ will return a set with as many Machine Learning models as the number of values present in the `runtime` variable
	- **It is assumed that all the models stored in `runtime` have the same type**
		- They all take the **same arguments, in the same order**
- ```TO```
	- Its argument ```output``` is a variable
	- `output` can be seen as the variable where the result of $RunML(i, GetML(rt))$ is stored
		- `output` will store the outputs of all the Machine Learning models in the set returned by $GetML(rt)$, when they get passed the inputs in $i$

##  High-level overview of how $GetML(rt)$ can be implemented

![[MLSchemaNotation.png]]
- For each runtime  in $rt$, each represented by a node of ```rdf:type``` ```mls:Run```
	- For each ```mls:hasOutput``` relation starting at the ```mls:Run``` node
		- If one of these relations lead to a node of ```rdf:type``` ```mls:Model```, then that node contains the ML model corresponding to the runtime

## Integration of the extension into the query plan
### Initial logical operator class
```rust
pub enum Term {
	Variable(String),
	Constant(u32),
}
```
```rust
pub type TriplePattern = (Term, Term, Term);
```
```rust
pub enum LogicalOperator {
	Scan {
		pattern: TriplePattern,
	},
	
	Selection {
		predicate: Box<LogicalOperator>,
		condition: Condition,
	},
	
	Projection {
		predicate: Box<LogicalOperator>,
		variables: Vec<String>,
	},
	
	Join {
		left: Box<LogicalOperator>,
		right: Box<LogicalOperator>,
	},
	
	Subquery {
		inner: Box<LogicalOperator>,
		projected_vars: Vec<String>,
	},

}
```

### Query plan representation
```rust 
LogicalOperator::MLRun(input: (LogicalOperator, HashMap<u32, Term>), mlmodels: LogicalOperator, output: Vec<String>)
```
- `input`
	- Represents the argument of the `RUN` component of the ML Clause
	- It contains a tuple
		- The first element of the tuple is the logical operator used to retrieve all the values of the variables passed as input to the ML models
		- The second element of the tuple is a HashMap, and is used as a representation for constants
			- *Key*: an integer which is the index representing the position at which the constant it is associated to is defined as argument to `ON`
			- *Value*: a `Term::Constant` used to represent a constant
- `mlmodels`
	- Represents the argument of the `ON` keyword
	- Its value is a `LogicalOperator` object that corresponds to the logical operator used to retrieve the unique or multiple machine learning models
- `output`
	- Represents the argument to the `TO` keyword
	- This variable will store the outputs of the Machine Learning models defined in `mlmodels`

### Query plan building process
#### Explanation of the query plan building algorithm
The query plan building algorithm is executed by the `build_logical_plan_optimized` function. Thus, this section explains what the code in this function does.

In the initial query building process, for each pattern in the `WHERE` clause, a `LogicalOperator::Scan` logical operator is created for each of them. All these operators are then stored in a vector called `scan_operators`. Before adding these Scan operators to the vector, if there are filters that can be pushed down onto these operators, `LogicalOperator::Filter` logical operators are applied to these Scan operators. The operators in this vector are then ordered in ascending order based on their cost. Thus, the least costly operator is at the beginning of `scan_operators` while the costliest operator is at the end of this vector. Then, we initialise a variable called `result` which takes as value the first element of `scan_operators`.  For each operator `op` in `scan_operators`, excluding the first one, a `LogicalOperator::Join` logical operator with as first argument `result` and as second argument `op` is assigned to `result`. After looping through `scan_operators`, the `LogicalOperator::Projection` logical operator is applied on `result` to project the values it will obtain onto the variables defined in `SELECT`.

To augment SPARQL with the new clause, the arguments of the ML Clause first needs to be extracted to a triple `mlparams`, whose first element, `features`, a Vector of Strings, corresponds to the argument of the `RUN` keyword. Its second element, `run_str`, corresponds to the argument of the `ON` keyword while `output_str` corresponds to the argument of the `TO` keyword.

Then, the ML models are retrieved from the runtimes passed as argument to `ON`. For this, the algorithm, described in the section [[#High-level overview of how $GetML(rt)$ can be implemented]], is applied. The algorithm can be defined as the SPARQL query below, which retrieves from the knowledge graph all the nodes corresponding to runtimes, and the models associated to them. It is assumed for this case that the argument passed to the `ON` keyword of ML Clause is defined to be a variable named `?r`.
```sparql 
SELECT ?ml, ?r
WHERE{
	?r rdf:type mls:Run.
	?r mls:hasOutput ?ml.
	?ml rdf:type mls:Model.
} 
``` 
This chain of logical operators for the above SPARQL query involves applying a `LogicalOperator::Projection` on the `?ml` and `?r` variables of a Join which has as its first parameter another `LogicalOperator::Join` and as its second parameter the `LogicalOperator::Scan` on the `?r mls:hasOutput ?ml.` pattern. This inner Join has as its first parameter a Scan on the `?r rdf:type mls:Run.` pattern, and as its second parameter a Scan on the `?ml rdf:type mls:Model.` pattern . This `LogicalOperator::Projection`, which retrieves all the values of the `?ml`, `?r` variables when executed, is passed as the second argument of the `LogicalOperator::MLRun` logical operator. The `?ml` variable correspond to the $ml$ argument of $GetML$, which represents all the Machine Learning models to which the inputs $i$ need to be passed. Meanwhile, `?r` represents the set of runtimes from which the Machine Learning models stored in `?ml` are deduced. 

However, if the argument passed to the `ON` keyword of ML Clause is defined to be a constant `someRTCstName`, then instead of the Projection on the `?ml` and `?r` variables, we just project on the `?ml` variable. Thus, the query that will be executed in this case with the chain of logical operators nested inside one another will be as follows:
```sparql 
SELECT ?ml
WHERE{
	someRTCstName rdf:type mls:Run.
	someRTCstName mls:hasOutput ?ml.
	?ml rdf:type mls:Model.
} 
``` 
The logical operator representation of any of these two queries above, depending on if the argument of the `ON` keyword is a variable or not, corresponds to the second argument of the `LogicalOperator::MLRun` logical operator, called `mlmodels`.

The logical operator to retrieve the ML models is stored in a variable called `runselect`. Thus, once `runselect` and `mlparams` are assigned a value, the logical operator stored in the `result` variable is "unpacked" until we reach a `LogicalOperator::Scan` logical operator. This means that as `result` is of type `LogicalOperator::Projection`, we first take the `predicate` argument of the `LogicalOperator::Projection` operator, which is itself another logical operator, and assign this value to the `initial` variable. This operator is either a `LogicalOperator::Join` or a `LogicalOperator::Selection`. If this operator is of type `LogicalOperator::Join`, nothing needs to be done. However, if the operator is a `LogicalOperator::Selection`, we take its `predicate` argument to get a `LogicalOperator::Join` logical operator. For the sake of simplicity, let us call the operation explained with the preceding two sentences Join Extraction. 

For each variable `varble`  present in the `features` argument of `mlparams`, we first start with the logical operator we got after executing our first Join Extraction, and assign the obtained value to the variable `initial`. Then, we continue executing Join Extractions on `initial` until we reach a  `LogicalOperator::Join` operator that has as its `right` argument a  `LogicalOperator::Scan` operator which scans a pattern where `varble` is present, either as subject, object or predicate. The number of Join Extractions that had to be executed in order to reach this state are recorded. For each `varble`, we store its String representation in a Vector called `varVec`.

The variable for which the least amount of Join Extraction operations had to be executed has its Join Extraction execution count stored in a variable called `maxdepth`. With the help of this variable, we apply a `LogicalOperator::Projection` with as `predicate` argument the logical operator nested at the level stored in that `maxdepth` variable, and as `variables` argument the vector `varVec`. Let this logical operator be called `fstArg`.

Now, for each constant `cstnt` present in the `features` argument of `mlparams`, we create a `Term::Constant` object that represents it. This object is then added to a HashMap called `cstMap`,  and associated with an integer key that indicates the position of the constant within `features`.

The first argument of the logical operator `LogicalOperator::MLRun` for the ML Clause is a tuple containing firstly `fstArg` and secondly the `cstMap` HashMap. This argument is called `input`

The third argument of the logical operator `LogicalOperator::MLRun` for the ML Clause is a vector with a single string which indicates the variable where the outputs of the single or multiple Machine Learning models should be stored. This argument is called `output`.

Once the `LogicalOperator::MLRun` has been created, the `LogicalOperator::Join`, nested in `result` a level above the one indicated by `maxdepth`, has its `left` argument replaced by a `LogicalOperator::Join`, which joins this `left` argument that is about to be replaced with the `LogicalOperator::MLRun`.

#### Pseudocode of the query plan building algorithm
1. `mlparams = (features: Vec<&str>, run_str: &str, output_str: &str)` represents the arguments retrieved from the SPARQL declaration of the ML Clause
2. Get the query plan representation of getting all the nodes of `rdf:type` `mls:Model` as `runselect`: 
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
3. For each `(subject_str: &str, predicate_str: &str, object_str: &str)` in `patterns`
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
4. Sort operators in `scan_operators` by cost
5. Let `result` be the first element of `scan_operators` 
6. For each `op` in `scan_operators`, excluding the first element
	1. `result = LogicalOperator::Join(result, op)`
7. Let `varNames` be the Vector containing the names of the variables given as parameter to the `SELECT` clause of SPARQL
8. `result = LogicalOperator::Projection(result, varNames)`
9. `initial = result.predicate`
10. Set the value of `maxdepth` to 0
11. Initialise an empty Vector `varVec`
12. For each `feature` in `mlparams.features` that is a variable
	1. Set the value of `depth` to the size of `scan_operators` 
	2. Convert `feature` to a String `fstr`
	3. Add `fstr` to `varVec`
	4. While `initial` is not of type `LogicalOperator::Scan`
		1. If `initial` is of type `LogicalOperator::Selection`
			1. `initial = initial.predicate`
		2. If `initial.right` scans a pattern containing `feature`
			1. If `depth > maxdepth`
				1. `maxdepth = depth`
			2. Go to the next iteration of the outer for loop
		3. `depth -= 1`
		4. `initial = initial.left`
	5. `initial = result.predicate`
13. Initialise an empty `cstMap` HashMap, which will have integer keys and `Term::Constant` values
14. For each `feature` in `mlparams.features` that is a constant
	1. Let `cterm` be the `Term::Constant` associated to `feature`
	2. Add `cterm` to `cstMap` with as key the index at which they were present in `features`
15. Set the value of `startDepth` to the size of `scan_operators`
16. `initPos = result.predicate`
17. If `initPos` is of type `LogicalOperator::Selection`
	1. `initial = initial.predicate`
18. While `startDepth != maxDepth`
	1. If `initPos` is of type `LogicalOperator::Selection`
		1. `initial = initial.predicate`
	2. `initPos = initPos.left`
	3. `startDepth -= 1`
19. `fstArg = LogicalOperator::Projection(initPos, varVec)`
20. Replace the `left` argument of the `LogicalOperator::Join` at nested level `maxdepth + 1` of `result` by `LogicalOperator::Join(initPos, LogicalOperator::MLRun((fstArg, cstMap), runselect, output_str))`
21. Return `result`

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
PhysicalOperator::MLExecute(input: (PhysicalOperator, HashMap<u32, Term>), mls: PhysicalOperator, output: Vec<String>)
```
- `input`: 
	- Also a tuple, just like the `input` argument of `LogicalOperator::MLRun`
		- The only difference between these tuples is that the tuple of `PhysicalOperator::MLExecute`has as its first element a `PhysicalOperator` representing the physical plan that needs to be done to retrieve all the values of the variables which will be passed as inputs to the ML models
- `mls`: physical plan representation of the ML model(s), deduced from the runtime(s)
- `output`: `Term::Variable` representation of the variable which is the argument of the `TO` keyword

### Converting the logical operator to a physical operator
This conversion from the logical operator to a physical operator is done within the `find_best_plan_recursive` function. The algorithm described below indicates only the extra condition that should be added to the `match` operator in the `find_best_plan_recursive`, while not deleting any of the existing code in the function
#### Pseudocode of the match extension needed to process the ML Clause
1. `logical_plan` is the `LogicalOperator::MLRun(input: (LogicalOperator, HashMap<u32, Term>), mlmodels: LogicalOperator, output: Vec<String>)`
2. `pplan = find_best_plan_recursive(logical_plan.input[0])`
3. `mlplan = find_best_plan_recursive(logical_plan.mlmodels)`
4. Return `PhysicalOperator::MLExecute((pplan, logical_plan.input[1]), mlplan, logical_plan.output)`
#### Explanation of the match extension needed to process the ML Clause
When `find_best_plan_recursive` encounters a logical operator of type `LogicalOperator::MLRun(input: (LogicalOperator, HashMap<u32, Term>), mlmodels: LogicalOperator, output: Vec<String>)`, it only needs to convert the first element of the `input` argument tuple and the `mlmodels` argument to physical operators, using `find_best_plan_recursive`, to convert the the logical operator of ML Clause into a physical operator. 

### Processing the physical operator
This physical operator execution is done within the `execute_with_ids` function. This function returns a Vector of HashMaps, which model a table. The HashMaps represent the rows of the table. For each element in the HashMap, the key corresponds to the column name (which corresponds to a variable) in which the value mapped to it belongs to. 

Just like in the previous section, the algorithm described below only describes the extra condition that should be added to the `match` operator of `execute_with_ids` to execute the physical operator of ML Clause.
#### Pseudocode of the match extension needed to execute the ML Clause
1. `operator = PhysicalOperator::MLExecute(input: (PhysicalOperator, HashMap<u32, Term>), mls: PhysicalOperator, output: Vec<String>)`
2. `inputToML = execute_with_ids(operator.input[0])`
3. For each `(k, v)` pair in `operator.input[1]`
	1. Create a column at the `k`-th position of `inputToML`
		1. Get a string `dbstr` corresponding to the `Term::Constant`
		2. Each row of the column takes the number stored in the `Term::Constant`
		3. Name the column `dbstr`
4. `MLmodels = execute_with_ids(operator.mls)`
5. Initialise an empty table `restable` with a column for each variable in `output` and no row yet
6. For each row `r` of `inputToML`
	1. For each model `mlid` in `MLmodels`
		1. Retrieve the ML model `mlmod` associated with id `mlid` in the database
		2. Pass the values corresponding to the ids stored in `r` at once to `mlmod`
		3. Add these `mlmod` outputs to the database
		4. Add a new row to `restable` where we store the database ids of the outputs of `ml`
7. Return `restable`

#### Explanation of the pseudocode
In the case `execute_with_ids` encounters a physical operator of type `PhysicalOperator::MLExecute`, it first processes the first element of the tuple given as the `input` argument to get a table `inputToML`. This table will have a column for each of the variables passed as input to the `RUN` keyword of ML Clause. The values stored in the table are the ids real values that are stored in the database.

Then, the second element of the `input` argument tuple of the physical operator is retrieved. This second argument is a HashMap with an integer key and a `Term::Constant` value. For each `(k, v)` key-value pair in the HashMap, we create a row in `inputToML` with as name the String `dbstr` associated with `v`. For each row cell of the column we place in it the integer stored in the `Term::Constant` object `v`. This integer indicates the id of `v` in the database.

Then, for each row `r` of `inputToML`, for each index `mlid` from `MLmodels`, which is a table with a single column, we first get the ML model `mlmod` corresponding to `mlid` in the database. Then, we pass all the values corresponding to the ids stored in `r` to `mlmod`. Then, the outputs of `mlmod` are added to the database, and their corresponding ids are retrieved. A row with these ids are then added to a table called `restable`. This table will later be returned.


## Query execution example
### Query

```sparql
SELECT ?building ?energyPrediction WHERE {  
   ?building hvac:temperature ?temp ;  
             hvac:humidity ?humid ;  
             hvac:occupancy ?occ ;  
             hvac:sunlight ?sun ;  
             hvac:windSpeed ?wind ;  
             time:hour ?hour ;  
             time:dayOfWeek ?day .  
   RUN {?temp, ?humid, ?occ, ?sun, ?wind, ?hour, ?day} ON ?runtime TO ?energyPrediction  
}
```

