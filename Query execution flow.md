
## `execute_query_rayon_parallel2_volcano(sparql: &str, database: &mut SparqlDatabase, ) -> Vec<Vec<String>>` 
1. `normalize_query(sparql: &str) -> &str` 
	- Normalises the query by removing the `SELECT` keyword
	```rust 
	let sparql = normalize_query(sparql);
	```
2. Register prefixes defined in the query with `database.register_prefixes_from_query(&sparql);` 
3. Parse results with `parse_sparql_query` to get:
	- **insert_clause**: elements to be added to the database from `INSERT`
	- **variables**: variables and the values assigned to them
	- **patterns**: `(subject, object, predicate)` triples that indicate what to query
	- **filters**: conditions on which to filter data
	- **parsed_prefixes**: map with as keys prefix names and as values URIs
	- **values_clauses**: arguments of `VALUES`
	- **binds**: arguments of `BIND`
	- **subqueries**: vector of subqueries
	- **limit**: given by `LIMIT`, specifies how many values should be returned
	- **order_conditions**: variables and how they should be ordered
4. Prefixes stored in the database are added to `parsed_prefixes`
	```rust 
	let mut prefixes = parsed_prefixes;
	database.share_prefixes_with(&mut prefixes);
	``` 
5. Insert elements into database from `insert_clause` with `process_insert_clause`
	```rust 
	process_insert_clause(insert_clause, database);
	```
6. If `SELECT *`  used, gather all the variables from the patterns
	```rust 
	if variables == vec![("*", "*", None)] {
		let mut all_vars = BTreeSet::new();
		for (subject_var, _, object_var) in &patterns {
			all_vars.insert(*subject_var);
			all_vars.insert(*object_var);
		}
		variables = all_vars.into_iter().map(|var| ("VAR", var, None)).collect();
	}
	```
7. Process variables for aggregation and those for selection with `process_variables`
	[[process_variables]]
	```rust 
	let mut selected_variables: Vec<(String, String)> = Vec::new();
	let mut aggregation_vars: Vec<(&str, &str, &str)> = Vec::new();
	process_variables(&mut selected_variables, &mut aggregation_vars, variables);
	```
8.  For each (subject, predicate, object) triple, replace the prefixes of subject, predicate and object by their corresponding URI
	```sparql
	let resolved_patterns: Vec<(&str, &str, &str)> = patterns
            .iter()
            .map(|(subject_var, predicate, object_var)| {
                let (resolved_subject, resolved_predicate, resolved_object) =
                    resolve_triple_pattern(subject_var, predicate, object_var, database, &prefixes);
                
                // Leak strings to get 'static lifetime
                let subject_static: &'static str = Box::leak(resolved_subject.into_boxed_str());
                let predicate_static: &'static str = Box::leak(resolved_predicate.into_boxed_str());
                let object_static: &'static str = Box::leak(resolved_object.into_boxed_str());
                
                (subject_static, predicate_static, object_static)
            })
            .collect();

	 ```
# Phase 1
	 