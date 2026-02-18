# Glyph Equivalence Classes Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Reduce DFS branching factor by grouping metrically identical glyphs into equivalence classes, searching over representatives, and expanding matches.

**Architecture:** Add `EquivClasses` computation to `dfs.rs`, deduplicate `constraint.state_allowed` before the DFS, expand representative-only matches into all class-member variants. No Python-side changes.

**Tech Stack:** Rust (solver_rs), no new dependencies

---

### Task 1: Add EquivClasses struct and compute_equiv_classes

**Files:**
- Modify: `solver_rs/src/dfs.rs`

**Step 1: Write the failing test**

Add to the bottom of `solver_rs/src/dfs.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// Helper: build a WidthTable from flat data.
    fn make_wt(charset: &str, table: &[f64], left: &[f64], right: &[f64]) -> WidthTable {
        WidthTable {
            charset: charset.chars().collect(),
            width_table: table.to_vec(),
            left_edge: left.to_vec(),
            right_edge: right.to_vec(),
            n: charset.len(),
        }
    }

    #[test]
    fn test_equiv_classes_with_duplicates() {
        // charset "abc" where b(1) and c(2) are equivalent:
        // same column, same row, same left_edge, same right_edge
        let wt = make_wt(
            "abc",
            &[
                // row a (prev=a): next=a,b,c
                5.0, 6.0, 6.0,
                // row b (prev=b): next=a,b,c  ← same as row c
                5.0, 6.0, 6.0,
                // row c (prev=c): next=a,b,c  ← same as row b
                5.0, 6.0, 6.0,
            ],
            &[4.0, 5.0, 5.0],   // left_edge: b == c
            &[0.0, 1.0, 1.0],   // right_edge: b == c
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        assert_eq!(equiv.num_classes, 2, "expected 2 classes: {{a}}, {{b,c}}");
        assert_ne!(equiv.class_of[0], equiv.class_of[1], "a should be in different class than b");
        assert_eq!(equiv.class_of[1], equiv.class_of[2], "b and c should be in same class");
        // b (idx 1) should be representative (first in class)
        let bc_class = equiv.class_of[1];
        assert_eq!(equiv.members[bc_class][0], 1, "b should be representative");
        assert_eq!(equiv.members[bc_class].len(), 2);
    }

    #[test]
    fn test_equiv_classes_all_unique() {
        let wt = make_wt(
            "abc",
            &[
                5.0, 6.0, 7.0,
                8.0, 9.0, 10.0,
                11.0, 12.0, 13.0,
            ],
            &[4.0, 5.0, 6.0],
            &[0.0, 1.0, 2.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        assert_eq!(equiv.num_classes, 3);
        for i in 0..3 {
            assert_eq!(equiv.members[equiv.class_of[i]].len(), 1);
        }
    }

    #[test]
    fn test_equiv_classes_with_extra_columns() {
        // b and c have same width properties but different extra column → not equivalent
        let wt = make_wt(
            "abc",
            &[
                5.0, 6.0, 6.0,
                5.0, 6.0, 6.0,
                5.0, 6.0, 6.0,
            ],
            &[4.0, 5.0, 5.0],
            &[0.0, 1.0, 1.0],
        );
        let extra = vec![10.0, 20.0, 30.0]; // different for b(20) and c(30)
        let equiv = compute_equiv_classes(&wt, &[&extra]);
        assert_eq!(equiv.num_classes, 3, "extra column should split b and c");
    }
}
```

**Step 2: Run test to verify it fails**

Run: `cargo test -p unredact-solver -- test_equiv_classes 2>&1`
Expected: compilation error — `compute_equiv_classes` not defined

**Step 3: Write minimal implementation**

Add to `solver_rs/src/dfs.rs`, after the existing `use` statements (there are none currently, so add at the top):

```rust
use std::collections::HashMap;
```

Add the struct and function after `compute_length_bounds` (before the `#[cfg(test)]` block):

```rust
/// Equivalence classes of metrically identical characters.
pub struct EquivClasses {
    /// class_of[char_idx] -> class_id
    pub class_of: Vec<usize>,
    /// members[class_id] -> Vec<char_idx> (first element is the representative)
    pub members: Vec<Vec<usize>>,
    /// Number of equivalence classes (K <= N)
    pub num_classes: usize,
}

/// Compute equivalence classes from width table properties.
///
/// Two characters are equivalent when they have identical:
/// - column in width_table (incoming advance from any predecessor)
/// - row in width_table (outgoing advance to any successor)
/// - left_edge and right_edge values
/// - values in any extra columns (e.g. space_advance for full_name mode)
///
/// Uses exact float comparison via f64::to_bits() — FreeType values
/// originate from 26.6 fixed-point, exactly representable as f64.
pub fn compute_equiv_classes(wt: &WidthTable, extra: &[&[f64]]) -> EquivClasses {
    let n = wt.n;
    let sig_len = 2 * n + 2 + extra.len();

    let mut signatures: Vec<Vec<u64>> = Vec::with_capacity(n);
    for i in 0..n {
        let mut sig = Vec::with_capacity(sig_len);
        // Column: advance of char i when preceded by each char p
        for p in 0..n {
            sig.push(wt.advance(p, i).to_bits());
        }
        // Row: advance of each char f when preceded by char i
        for f in 0..n {
            sig.push(wt.advance(i, f).to_bits());
        }
        sig.push(wt.left_edge[i].to_bits());
        sig.push(wt.right_edge[i].to_bits());
        for col in extra {
            sig.push(col[i].to_bits());
        }
        signatures.push(sig);
    }

    let mut class_of = vec![0usize; n];
    let mut members: Vec<Vec<usize>> = Vec::new();
    let mut sig_to_class: HashMap<Vec<u64>, usize> = HashMap::new();

    for i in 0..n {
        if let Some(&class_id) = sig_to_class.get(&signatures[i]) {
            class_of[i] = class_id;
            members[class_id].push(i);
        } else {
            let class_id = members.len();
            sig_to_class.insert(signatures[i].clone(), class_id);
            class_of[i] = class_id;
            members.push(vec![i]);
        }
    }

    let num_classes = members.len();
    EquivClasses { class_of, members, num_classes }
}
```

**Step 4: Run test to verify it passes**

Run: `cargo test -p unredact-solver -- test_equiv_classes 2>&1`
Expected: 3 tests PASS

**Step 5: Commit**

```bash
git add solver_rs/src/dfs.rs
git commit -m "feat(solver): add EquivClasses struct and compute_equiv_classes"
```

---

### Task 2: Add dedup_state_allowed and expand_match functions

**Files:**
- Modify: `solver_rs/src/dfs.rs`

**Step 1: Write the failing tests**

Add to the `tests` module in `solver_rs/src/dfs.rs`:

```rust
    #[test]
    fn test_dedup_state_allowed() {
        // charset "abc", b and c equivalent → 2 classes
        let wt = make_wt(
            "abc",
            &[5.0,6.0,6.0, 5.0,6.0,6.0, 5.0,6.0,6.0],
            &[4.0, 5.0, 5.0],
            &[0.0, 1.0, 1.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        // Single state allowing all chars
        let constraint = Constraint {
            state_allowed: vec![vec![0, 1, 2]],
            state_next: vec![vec![0, 0, 0]],
            accept_states: vec![true],
        };
        let deduped = dedup_state_allowed(&equiv, &constraint);
        assert_eq!(deduped.len(), 1);
        assert_eq!(deduped[0].len(), 2, "should be [a_rep, bc_rep]");
        assert!(deduped[0].contains(&0)); // a
        assert!(deduped[0].contains(&1)); // b (representative of {b,c})
        assert!(!deduped[0].contains(&2)); // c excluded (b is representative)
    }

    #[test]
    fn test_dedup_capitalized_constraint() {
        // charset "aAbB", A and B equivalent (same widths), a and b equivalent
        let wt = make_wt(
            "aAbB",
            &[
                5.0, 8.0, 5.0, 8.0,  // row a == row b
                7.0, 10.0, 7.0, 10.0, // row A == row B
                5.0, 8.0, 5.0, 8.0,  // row b == row a
                7.0, 10.0, 7.0, 10.0, // row B == row A
            ],
            &[4.0, 6.0, 4.0, 6.0],
            &[0.0, 1.0, 0.0, 1.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        assert_eq!(equiv.num_classes, 2); // {a,b} and {A,B}
        // Capitalized: state 0 = uppercase [A(1), B(3)], state 1 = lowercase [a(0), b(2)]
        let constraint = Constraint {
            state_allowed: vec![vec![1, 3], vec![0, 2]],
            state_next: vec![vec![-1, 1, -1, 1], vec![1, -1, 1, -1]],
            accept_states: vec![false, true],
        };
        let deduped = dedup_state_allowed(&equiv, &constraint);
        assert_eq!(deduped[0].len(), 1, "state 0: one uppercase rep");
        assert_eq!(deduped[1].len(), 1, "state 1: one lowercase rep");
    }

    #[test]
    fn test_expand_match_singleton() {
        let wt = make_wt(
            "abc",
            &[5.0,6.0,7.0, 8.0,9.0,10.0, 11.0,12.0,13.0],
            &[4.0, 5.0, 6.0],
            &[0.0, 1.0, 2.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        let byte_to_idx = build_byte_to_idx(&wt.charset);
        // All singletons — expand "ab" → just ["ab"]
        let path: Vec<u8> = b"ab".to_vec();
        let expanded = expand_match(&path, &equiv, &wt.charset, &byte_to_idx);
        assert_eq!(expanded.len(), 1);
        assert_eq!(expanded[0], b"ab");
    }

    #[test]
    fn test_expand_match_multi_member() {
        let wt = make_wt(
            "abc",
            &[5.0,6.0,6.0, 5.0,6.0,6.0, 5.0,6.0,6.0],
            &[4.0, 5.0, 5.0],
            &[0.0, 1.0, 1.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        let byte_to_idx = build_byte_to_idx(&wt.charset);
        // b and c equivalent, representative is b
        // Path "ab" (DFS placed representative b at position 1)
        // Expand → "ab", "ac"
        let path: Vec<u8> = b"ab".to_vec();
        let mut expanded = expand_match(&path, &equiv, &wt.charset, &byte_to_idx);
        expanded.sort();
        assert_eq!(expanded.len(), 2);
        assert_eq!(expanded[0], b"ab");
        assert_eq!(expanded[1], b"ac");
    }

    #[test]
    fn test_expand_match_multi_position() {
        let wt = make_wt(
            "abc",
            &[5.0,6.0,6.0, 5.0,6.0,6.0, 5.0,6.0,6.0],
            &[4.0, 5.0, 5.0],
            &[0.0, 1.0, 1.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        let byte_to_idx = build_byte_to_idx(&wt.charset);
        // Path "abb" → expand both b positions
        // → "abb", "abc", "acb", "acc"
        let path: Vec<u8> = b"abb".to_vec();
        let mut expanded = expand_match(&path, &equiv, &wt.charset, &byte_to_idx);
        expanded.sort();
        assert_eq!(expanded.len(), 4);
        assert_eq!(expanded[0], b"abb");
        assert_eq!(expanded[1], b"abc");
        assert_eq!(expanded[2], b"acb");
        assert_eq!(expanded[3], b"acc");
    }
```

**Step 2: Run test to verify it fails**

Run: `cargo test -p unredact-solver -- test_dedup test_expand 2>&1`
Expected: compilation error — `dedup_state_allowed`, `expand_match`, `build_byte_to_idx` not defined

**Step 3: Write minimal implementation**

Add to the top of `solver_rs/src/dfs.rs` (alongside the existing HashMap import):

```rust
use std::collections::HashSet;
```

Add after `compute_equiv_classes` (before the `#[cfg(test)]` block):

```rust
/// Deduplicate state_allowed to one representative per equivalence class.
pub fn dedup_state_allowed(equiv: &EquivClasses, constraint: &Constraint) -> Vec<Vec<usize>> {
    constraint.state_allowed.iter().map(|allowed| {
        let mut seen = HashSet::new();
        allowed.iter().copied().filter(|&idx| {
            seen.insert(equiv.class_of[idx])
        }).collect()
    }).collect()
}

/// Build byte→charset_index lookup (ASCII only).
pub fn build_byte_to_idx(charset: &[char]) -> [usize; 128] {
    let mut lookup = [0usize; 128];
    for (i, &c) in charset.iter().enumerate() {
        lookup[c as usize] = i;
    }
    lookup
}

/// Expand a representative-only path into all class-member variants.
fn expand_match(
    path: &[u8],
    equiv: &EquivClasses,
    charset: &[char],
    byte_to_idx: &[usize; 128],
) -> Vec<Vec<u8>> {
    let mut results: Vec<Vec<u8>> = vec![Vec::with_capacity(path.len())];

    for &byte in path {
        let idx = byte_to_idx[byte as usize];
        let class_id = equiv.class_of[idx];
        let members = &equiv.members[class_id];

        if members.len() == 1 {
            for r in &mut results {
                r.push(byte);
            }
        } else {
            let mut new_results = Vec::with_capacity(results.len() * members.len());
            for existing in &results {
                for &member_idx in members {
                    let mut s = existing.clone();
                    s.push(charset[member_idx] as u8);
                    new_results.push(s);
                }
            }
            results = new_results;
        }
    }

    results
}
```

**Step 4: Run test to verify it passes**

Run: `cargo test -p unredact-solver -- test_dedup test_expand 2>&1`
Expected: all 5 tests PASS

**Step 5: Commit**

```bash
git add solver_rs/src/dfs.rs
git commit -m "feat(solver): add dedup_state_allowed, build_byte_to_idx, expand_match"
```

---

### Task 3: Wire equivalence classes into DFS and solve_subtree

**Files:**
- Modify: `solver_rs/src/dfs.rs`

This is the core change: modify `solve_subtree` to accept equivalence class data, and modify `dfs` to use `deduped_allowed` for iteration and `expand_match` for result generation.

**Step 1: Write the failing integration test**

Add to the `tests` module in `solver_rs/src/dfs.rs`:

```rust
    #[test]
    fn test_solve_subtree_expands_equivalences() {
        // charset "abc", b(1) and c(2) equivalent
        let wt = make_wt(
            "abc",
            &[5.0,6.0,6.0, 5.0,6.0,6.0, 5.0,6.0,6.0],
            &[4.0, 5.0, 5.0],
            &[0.0, 1.0, 1.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        let constraint = Constraint {
            state_allowed: vec![vec![0, 1, 2]],
            state_next: vec![vec![0, 0, 0]],
            accept_states: vec![true],
        };
        let deduped = dedup_state_allowed(&equiv, &constraint);

        // target = 11.0, tol = 0.5, length 2
        // "ab": left[a]=4 + wt[a][b]=6 + right[b]=1 = 11 ✓
        // "ac": left[a]=4 + wt[a][c]=6 + right[c]=1 = 11 ✓ (equivalent to ab)
        let results = solve_subtree(
            &wt, 11.0, 0.5, 2, 2,
            "", 0.0, -1,
            &constraint, 0,
            &equiv, &deduped,
        );
        let mut texts: Vec<String> = results.iter().map(|r| r.text.clone()).collect();
        texts.sort();
        assert!(texts.contains(&"ab".to_string()), "should contain ab");
        assert!(texts.contains(&"ac".to_string()), "should contain ac (expanded from ab)");
    }

    #[test]
    fn test_solve_subtree_no_equiv_same_results() {
        // All unique chars — results should be identical to non-optimized
        let wt = make_wt(
            "abc",
            &[5.0, 6.0, 7.0, 8.0, 9.0, 10.0, 11.0, 12.0, 13.0],
            &[4.0, 5.0, 6.0],
            &[0.0, 1.0, 2.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        assert_eq!(equiv.num_classes, 3); // all singletons
        let constraint = Constraint {
            state_allowed: vec![vec![0, 1, 2]],
            state_next: vec![vec![0, 0, 0]],
            accept_states: vec![true],
        };
        let deduped = dedup_state_allowed(&equiv, &constraint);

        // target = 15.0, tol = 1.0, length 2
        let results = solve_subtree(
            &wt, 15.0, 1.0, 2, 2,
            "", 0.0, -1,
            &constraint, 0,
            &equiv, &deduped,
        );
        // Manually verify some expected matches
        for r in &results {
            let err = (r.width - 15.0).abs();
            assert!(err <= 1.0, "result {} has error {} > tolerance", r.text, err);
        }
    }
```

**Step 2: Run test to verify it fails**

Run: `cargo test -p unredact-solver -- test_solve_subtree 2>&1`
Expected: compilation error — solve_subtree signature doesn't match (missing equiv/deduped params)

**Step 3: Modify solve_subtree and dfs**

Update the `solve_subtree` function signature in `solver_rs/src/dfs.rs` to accept equiv and deduped_allowed:

```rust
pub fn solve_subtree(
    wt: &WidthTable,
    target: f64,
    tolerance: f64,
    min_length: usize,
    max_length: usize,
    prefix: &str,
    prefix_width: f64,
    last_char_idx: i32,
    constraint: &Constraint,
    start_state: usize,
    equiv: &EquivClasses,
    deduped_allowed: &[Vec<usize>],
) -> Vec<SolveResult> {
    let smax = compute_state_max(wt, constraint);
    let byte_to_idx = build_byte_to_idx(&wt.charset);
    let mut results = Vec::new();

    if prefix.is_empty() {
        for &first_idx in &deduped_allowed[start_state] {
            let start_width = wt.left_edge[first_idx];
            if start_width > target + tolerance {
                continue;
            }
            let ns = constraint.state_next[start_state][first_idx];
            if ns < 0 {
                continue;
            }
            if max_length > 1 {
                let max_possible = start_width + smax[ns as usize] * (max_length as f64 - 1.0);
                if max_possible + tolerance < target {
                    continue;
                }
            }
            let mut path = vec![wt.charset[first_idx] as u8];
            dfs(
                wt, target, tolerance, min_length, max_length,
                constraint, &smax, equiv, deduped_allowed, &byte_to_idx,
                1, start_width, first_idx, &mut path, ns as usize,
                0, &mut results,
            );
        }
    } else {
        let mut path: Vec<u8> = prefix.bytes().collect();
        dfs(
            wt, target, tolerance, min_length, max_length,
            constraint, &smax, equiv, deduped_allowed, &byte_to_idx,
            0, prefix_width, last_char_idx as usize, &mut path, start_state,
            prefix.len(), &mut results,
        );
    }

    results
}
```

Update the `dfs` function to use `deduped_allowed` for iteration and `expand_match` for results:

```rust
#[inline(never)]
fn dfs(
    wt: &WidthTable,
    target: f64,
    tolerance: f64,
    min_length: usize,
    max_length: usize,
    constraint: &Constraint,
    smax: &[f64],
    equiv: &EquivClasses,
    deduped_allowed: &[Vec<usize>],
    byte_to_idx: &[usize; 128],
    depth: usize,
    acc_width: f64,
    last_idx: usize,
    path: &mut Vec<u8>,
    cstate: usize,
    prefix_len: usize,
    results: &mut Vec<SolveResult>,
) {
    let current_length = prefix_len + depth;

    if current_length >= min_length && constraint.accept_states[cstate] {
        let final_width = acc_width + wt.right_edge[last_idx];
        let err = (final_width - target).abs();
        if err <= tolerance {
            let expanded = expand_match(path, equiv, &wt.charset, byte_to_idx);
            for text_bytes in expanded {
                let text = unsafe { String::from_utf8_unchecked(text_bytes) };
                results.push(SolveResult { text, width: final_width, error: err });
            }
        }
    }

    if current_length >= max_length {
        return;
    }

    let chars_left = max_length - current_length;

    for &next_idx in &deduped_allowed[cstate] {
        let advance = wt.advance(last_idx, next_idx);
        let new_width = acc_width + advance;

        if new_width > target + tolerance {
            continue;
        }

        let ns = constraint.state_next[cstate][next_idx];
        if ns < 0 {
            continue;
        }

        if chars_left > 1 {
            let max_possible = new_width + smax[ns as usize] * (chars_left as f64 - 1.0);
            if max_possible + tolerance < target {
                continue;
            }
        }

        path.push(wt.charset[next_idx] as u8);
        dfs(
            wt, target, tolerance, min_length, max_length,
            constraint, smax, equiv, deduped_allowed, byte_to_idx,
            depth + 1, new_width, next_idx, path, ns as usize,
            prefix_len, results,
        );
        path.pop();
    }
}
```

**Step 4: Run test to verify it passes**

Run: `cargo test -p unredact-solver -- test_solve_subtree 2>&1`
Expected: all solve_subtree tests PASS

Note: `main.rs` and `full_name.rs` will have compilation errors at this point because they call `solve_subtree` with the old signature. That's expected — we fix those in Tasks 4 and 5.

**Step 5: Commit**

```bash
git add solver_rs/src/dfs.rs
git commit -m "feat(solver): wire equivalence classes into DFS and solve_subtree"
```

---

### Task 4: Wire into prefix generation and handle_solve

**Files:**
- Modify: `solver_rs/src/main.rs`

**Step 1: Update generate_prefixes to accept deduped_allowed**

Change the `generate_prefixes` function signature and inner `expand` function to use `deduped_allowed` instead of `constraint.state_allowed`:

```rust
fn generate_prefixes(
    wt: &WidthTable, target: f64, tolerance: f64, depth: usize,
    constraint: &Constraint, deduped_allowed: &[Vec<usize>],
) -> Vec<(String, f64, i32, usize)> {
    let mut prefixes = Vec::new();

    fn expand(
        wt: &WidthTable, target: f64, tolerance: f64,
        constraint: &Constraint, deduped_allowed: &[Vec<usize>],
        pfx: &mut Vec<u8>, acc_width: f64, last_idx: i32, remaining: usize, cstate: usize,
        prefixes: &mut Vec<(String, f64, i32, usize)>,
    ) {
        if remaining == 0 {
            let text = unsafe { String::from_utf8_unchecked(pfx.clone()) };
            prefixes.push((text, acc_width, last_idx, cstate));
            return;
        }
        for &next_idx in &deduped_allowed[cstate] {
            let advance = if last_idx == -1 {
                wt.left_edge[next_idx]
            } else {
                wt.advance(last_idx as usize, next_idx)
            };
            let new_width = acc_width + advance;
            if new_width > target + tolerance { continue; }
            let ns = constraint.state_next[cstate][next_idx];
            if ns < 0 { continue; }
            pfx.push(wt.charset[next_idx] as u8);
            expand(wt, target, tolerance, constraint, deduped_allowed, pfx, new_width, next_idx as i32, remaining - 1, ns as usize, prefixes);
            pfx.pop();
        }
    }

    expand(wt, target, tolerance, constraint, deduped_allowed, &mut Vec::new(), 0.0, -1, depth, 0, &mut prefixes);
    prefixes
}
```

**Step 2: Update handle_solve to compute equiv classes and pass them through**

In `handle_solve`, after building `wt` and `constraint`, add equiv computation and pass to `generate_prefixes` and `solve_subtree`:

```rust
// After constraint is built (line ~130):
let equiv = dfs::compute_equiv_classes(&wt, &[]);
let deduped_allowed = dfs::dedup_state_allowed(&equiv, &constraint);

// Update generate_prefixes call:
let prefixes = generate_prefixes(&wt, req.target, req.tolerance, prefix_depth, &constraint, &deduped_allowed);

// Update solve_subtree call inside par_iter:
let mut results = solve_subtree(
    &wt, req.target, req.tolerance,
    min_length, max_length,
    prefix, *prefix_width, *last_idx,
    &constraint, *pfx_state,
    &equiv, &deduped_allowed,
);
```

Add the necessary imports at the top of main.rs — update the existing `use dfs::` line to include the new public items:

```rust
use dfs::{Constraint, SolveResult, WidthTable, compute_length_bounds, solve_subtree,
          compute_equiv_classes, dedup_state_allowed};
```

**Step 3: Verify compilation**

Run: `cargo check -p unredact-solver 2>&1`
Expected: only `full_name.rs` errors remain (it still uses old solve_subtree signature)

**Step 4: Commit**

```bash
git add solver_rs/src/main.rs
git commit -m "feat(solver): wire equivalence classes into prefix generation and handle_solve"
```

---

### Task 5: Wire into full_name.rs

**Files:**
- Modify: `solver_rs/src/full_name.rs`

**Step 1: Update solve_full_name to compute equiv classes**

In `solve_full_name`, after building wt1, wt2, and constraint:

```rust
// Compute equiv classes — wt1 with extra columns for space/left_after_space
let equiv1 = compute_equiv_classes(&wt1, &[space_advance, left_after_space]);
let equiv2 = compute_equiv_classes(&wt2, &[]);
let deduped1 = dedup_state_allowed(&equiv1, &constraint);
let deduped2 = dedup_state_allowed(&equiv2, &constraint);
let byte_to_idx = build_byte_to_idx(word_charset);
```

Update the import line at the top of `full_name.rs`:

```rust
use crate::dfs::{Constraint, SolveResult, WidthTable, compute_length_bounds, solve_subtree,
                 EquivClasses, compute_equiv_classes, dedup_state_allowed, build_byte_to_idx};
```

**Step 2: Update first_allowed/body_allowed to use deduped versions**

Replace the extraction of `first_allowed` and `body_allowed` with deduped versions for the DFS (keep the original full lists for bounds computation):

```rust
// Keep full lists for bounds computation (lines 88-93 unchanged):
let first_allowed = &constraint.state_allowed[0];
let body_allowed = if constraint.state_allowed.len() > 1 {
    &constraint.state_allowed[1]
} else {
    &constraint.state_allowed[0]
};

// ... bounds computation using first_allowed, body_allowed (unchanged) ...

// Deduped lists for DFS iteration:
let first_deduped = &deduped1[0];
let body_deduped = if deduped1.len() > 1 {
    &deduped1[1]
} else {
    &deduped1[0]
};
```

**Step 3: Update dfs_first signature and iteration**

Add `equiv1`, `byte_to_idx`, `first_deduped`, `body_deduped`, `equiv2`, `deduped2` parameters to `dfs_first`. Change the iteration loop to use deduped lists:

```rust
let allowed = if depth == 0 { first_deduped } else { body_deduped };
```

Update the match handling in dfs_first to expand the first word before combining with second word results. Update the solve_subtree call to pass `&equiv2` and `&deduped2`:

```rust
if depth >= min_word_len && !is_start {
    let first_width = acc_width + wt1.right_edge[last_idx];
    let sp = space_advance[last_idx];
    let remaining = target - first_width - sp;

    if remaining > 0.0 {
        let (auto_min2, auto_max2) = compute_length_bounds(wt2, remaining, tolerance);
        let auto_min2 = auto_min2.max(min_word_len);

        if auto_min2 <= auto_max2 {
            let second_results = solve_subtree(
                wt2, remaining, tolerance,
                auto_min2, auto_max2,
                "", 0.0, -1,
                constraint, 0,
                equiv2, deduped2,
            );

            // Expand first word (all class-member variants)
            let first_expanded = crate::dfs::expand_match_pub(path, equiv1, charset, byte_to_idx);
            for first_bytes in &first_expanded {
                let first_text = unsafe { String::from_utf8_unchecked(first_bytes.clone()) };
                for r2 in &second_results {
                    let full_text = format!("{} {}", first_text, r2.text);
                    if !seen.contains(&full_text) {
                        let full_width = first_width + sp + r2.width;
                        let err = (full_width - target).abs();
                        if err <= tolerance {
                            seen.insert(full_text.clone());
                            results.push(SolveResult {
                                text: full_text,
                                width: full_width,
                                error: err,
                            });
                        }
                    }
                }
            }
        }
    }
}
```

Note: `expand_match` is currently private (`fn`, not `pub fn`). To use it from `full_name.rs`, either:
- Make it `pub fn` and rename to `expand_match` (simplest)
- Add a `pub` wrapper function

**Make expand_match public** in `dfs.rs` — change `fn expand_match` to `pub fn expand_match`.

**Step 4: Verify compilation and tests**

Run: `cargo check -p unredact-solver 2>&1`
Expected: clean compilation

Run: `cargo test -p unredact-solver 2>&1`
Expected: all tests PASS

**Step 5: Commit**

```bash
git add solver_rs/src/dfs.rs solver_rs/src/full_name.rs
git commit -m "feat(solver): wire equivalence classes into full_name solver"
```

---

### Task 6: End-to-end integration test

**Files:**
- Modify: `solver_rs/src/dfs.rs` (add test)

**Step 1: Write integration test comparing optimized vs reference results**

Add to the `tests` module in `solver_rs/src/dfs.rs`:

```rust
    #[test]
    fn test_solve_subtree_3char_expansion() {
        // charset "abc", b and c equivalent
        // Verify 3-char matches expand correctly
        let wt = make_wt(
            "abc",
            &[5.0,6.0,6.0, 5.0,6.0,6.0, 5.0,6.0,6.0],
            &[4.0, 5.0, 5.0],
            &[0.0, 1.0, 1.0],
        );
        let equiv = compute_equiv_classes(&wt, &[]);
        let constraint = Constraint {
            state_allowed: vec![vec![0, 1, 2]],
            state_next: vec![vec![0, 0, 0]],
            accept_states: vec![true],
        };
        let deduped = dedup_state_allowed(&equiv, &constraint);

        // target = 17.0, tol = 0.5, length 3
        // "abb": 4+6+6+1 = 17 → expands to abb, abc, acb, acc
        // "bab": 5+5+6+1 = 17 → expands to bab, bac, cab, cac
        let results = solve_subtree(
            &wt, 17.0, 0.5, 3, 3,
            "", 0.0, -1,
            &constraint, 0,
            &equiv, &deduped,
        );
        let mut texts: Vec<String> = results.iter().map(|r| r.text.clone()).collect();
        texts.sort();
        texts.dedup();

        // Verify all expected expansions are present
        for expected in &["abb", "abc", "acb", "acc", "bab", "bac", "cab", "cac"] {
            assert!(texts.contains(&expected.to_string()),
                "missing expected result: {}", expected);
        }

        // Verify all results have correct width
        for r in &results {
            assert!((r.width - 17.0).abs() <= 0.5,
                "result {} has bad width {}", r.text, r.width);
        }
    }
```

**Step 2: Run test**

Run: `cargo test -p unredact-solver -- test_solve_subtree_3char 2>&1`
Expected: PASS

**Step 3: Commit**

```bash
git add solver_rs/src/dfs.rs
git commit -m "test(solver): add integration test for 3-char equivalence expansion"
```

---

### Task 7: Build and smoke test the full server

**Files:** none (verification only)

**Step 1: Build release binary**

Run: `cargo build -p unredact-solver --release 2>&1`
Expected: clean build, no warnings

**Step 2: Run all tests**

Run: `cargo test -p unredact-solver 2>&1`
Expected: all tests PASS

**Step 3: Quick manual smoke test**

Start the server and send a test request:

```bash
# In one terminal:
cargo run -p unredact-solver --release

# In another:
curl -s -X POST http://127.0.0.1:3100/solve \
  -H "Content-Type: application/json" \
  -d '{
    "charset": "abcdefghijklmnopqrstuvwxyz",
    "width_table": [/* 676 values from a real font */],
    "left_edge": [/* 26 values */],
    "right_edge": [/* 26 values */],
    "target": 100.0,
    "tolerance": 5.0
  }'
```

If a real font width table is not readily available, the existing Python test suite can be used for end-to-end verification.

**Step 4: Commit (if any fixups needed)**

```bash
git add -A && git commit -m "fix(solver): address smoke test issues"
```

---

## Notes

- **No Python changes needed** — the API contract is unchanged
- **Result ordering may differ** — expanded results appear grouped by representative. If deterministic ordering matters, add a sort. The existing `results.sort_by(|a, b| a.error.partial_cmp(&b.error).unwrap())` in `handle_solve` already sorts by error.
- **Result limit interaction** — the atomic `total` counter in `handle_solve` already caps results. Expansion could temporarily exceed the limit within a single subtree batch, but this is bounded by the product of class sizes (typically small).
- **Potential future optimization** — if profiling shows expansion is a bottleneck for very large classes, consider lazy expansion or capping class size. Unlikely to matter in practice.
