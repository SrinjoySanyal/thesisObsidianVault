```rust
fn resolve_triple_pattern(
    subject_var: &str,
    predicate: &str,
    object_var: &str,
    database: &SparqlDatabase,
    prefixes: &HashMap<String, String>,
) -> (String, String, String) {
    if predicate == "RULECALL" {
        let rule_name = if subject_var.starts_with(':') {
            &subject_var[1..]
        } else {
            subject_var
        };
        let rule_key = rule_name.to_lowercase();

        // Look up the expanded predicate in the rule_map
        let expanded_rule_predicate = if let Some(expanded) = database.rule_map.get(&rule_key) {
            expanded.clone()
        } else {
            // Fallback: if the rule name is not already prefixed, prepend default prefix "ex"
            if let Some(default_uri) = prefixes.get("ex") {
                format!("{}{}", default_uri, rule_name)
            } else {
                rule_name.to_string()
            }
        };

        (
            object_var.to_string(),
            expanded_rule_predicate,
            "true".to_string(),
        )
    } else {
        // Resolve subject if it's not a variable
        let resolved_subject = if subject_var.starts_with('?') {
            subject_var.to_string()
        } else {
            database.resolve_query_term(subject_var, prefixes)
        };

        // For a normal triple pattern, resolve predicate and object
        let resolved_predicate = database.resolve_query_term(predicate, prefixes);

        // Resolve object if it's not a variable
        let resolved_object = if object_var.starts_with('?') {
            object_var.to_string()
        } else {
            database.resolve_query_term(object_var, prefixes)
        };

        (resolved_subject, resolved_predicate, resolved_object)
    }
} 
```