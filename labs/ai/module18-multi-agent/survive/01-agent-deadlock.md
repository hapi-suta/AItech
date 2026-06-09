# Survive 01: Agent Deadlock

Two agents wait for each other's output. The classifier needs the metric analyzer's enrichment before classifying. The metric analyzer needs the classifier's category before knowing which metrics to prioritize. Neither can proceed. The entire pipeline freezes for 45 minutes until a timeout kills both.

---

## The Disaster

On your **Mac terminal**, run the failure scenario:

```bash
python3 << 'PYEOF'
print("""
SCENARIO: Agent Deadlock

Your multi-agent pipeline:
  Monitor -> [Classifier + Metric Analyzer] -> Diagnostics -> Remediation

The deadlock:
  Classifier: "I need the enriched metrics before I can classify"
  Metric Analyzer: "I need the category to know which metrics to focus on"

  Both wait. Neither proceeds.

Timeline:
  09:00  Alert fires: "Something is wrong with pg-primary"
  09:00  Orchestrator sends alert to Classifier and Metric Analyzer
  09:00  Classifier requests enriched metrics from Metric Analyzer
  09:00  Metric Analyzer requests category from Classifier
  09:01  Both agents waiting... (deadlock)
  09:15  Alert queue starts backing up (30 alerts waiting)
  09:30  Monitoring shows 0 alerts processed in 30 minutes
  09:45  Timeout kills both agents. Pipeline restarts.
         45 minutes of alerts lost. 3 P1 alerts missed.

The deadlock was caused by a CIRCULAR DEPENDENCY:
  Classifier depends on Metric Analyzer
  Metric Analyzer depends on Classifier
  A -> B -> A (cycle)
""")

# Demonstrate the circular dependency
print("Circular Dependency Visualization:")
print("=" * 50)

dependencies = {
    "classifier": ["metric_analyzer"],       # classifier needs metric_analyzer
    "metric_analyzer": ["classifier"],       # metric_analyzer needs classifier
    "diagnostics": ["classifier", "metric_analyzer"],
    "remediation": ["diagnostics"],
}

print("\n  Agent Dependencies:")
for agent, deps in dependencies.items():
    print(f"    {agent} <- needs: {deps}")

# Detect cycle
def detect_cycle(deps):
    """
    Find circular dependencies using depth-first search.
    Returns the cycle if found, None if no cycle.
    """
    visited = set()          # nodes we've fully processed
    path = set()             # nodes in current exploration path

    def dfs(node, current_path):
        if node in path:
            return list(current_path) + [node]  # cycle found!
        if node in visited:
            return None

        path.add(node)
        current_path.append(node)

        for dep in deps.get(node, []):
            result = dfs(dep, current_path)
            if result:
                return result

        path.remove(node)
        current_path.pop()
        visited.add(node)
        return None

    for node in deps:
        result = dfs(node, [])
        if result:
            return result
    return None

cycle = detect_cycle(dependencies)
if cycle:
    cycle_str = " -> ".join(cycle)
    print(f"\n  CYCLE DETECTED: {cycle_str}")
else:
    print(f"\n  No cycle detected")

PYEOF
```

---

## Investigate

On your **Mac terminal**, find the root cause:

```bash
python3 << 'PYEOF'
print("Investigation: Why the Deadlock Happened")
print("=" * 55)

print("""
Root Cause Analysis:

1. CIRCULAR DEPENDENCY
   When the Metric Analyzer was updated to "category-aware metrics"
   (prioritize CPU metrics for performance, disk for storage),
   a developer added a dependency on the Classifier's output.

   Before (no cycle):
     Classifier -> Metric Analyzer -> Diagnostics
     (Classifier runs first, produces category)
     (Metric Analyzer uses category to prioritize metrics)

   After (cycle!):
     Classifier -> Metric Analyzer -> Classifier
     (Classifier wants enriched metrics before classifying)
     (Metric Analyzer wants category before enriching)

2. NO TIMEOUT ON AGENT REQUESTS
   When an agent requests data from another agent,
   there was no timeout. It waited forever.

3. NO DEPENDENCY VALIDATION
   The pipeline didn't check for cycles when agents registered.
   The circular dependency was invisible until runtime.

4. NO DEADLOCK DETECTION
   No monitor checked for "agent waiting too long."
   The 45-minute timeout was on the ENTIRE pipeline,
   not on individual agent waits.
""")

# Show the before and after dependency graphs
print("Before (working):")
print("  Monitor -> Classifier -> Metric Analyzer -> Diagnostics")
print("  No cycles. Each agent runs in order.")

print("\nAfter (broken):")
print("  Monitor -> [Classifier <-> Metric Analyzer] -> Diagnostics")
print("  Cycle between Classifier and Metric Analyzer!")
print("  Neither can start because both are waiting.")

PYEOF
```

---

## The Fix

On your **Mac terminal**, run the fix:

```bash
python3 << 'PYEOF'
import time

print("""
FIX: Four layers of deadlock prevention.

Layer 1: Dependency validation (reject cycles at startup)
Layer 2: Request timeouts (don't wait forever)
Layer 3: Async processing (don't block, use callbacks)
Layer 4: Deadlock detection (monitor wait times)
""")

print("Layer 1: Dependency Validation")
print("=" * 50)

def validate_dependencies(deps):
    """
    Check for cycles BEFORE starting the pipeline.
    If a cycle is found, refuse to start.

    DBA analogy: like checking for foreign key cycles
    before creating tables. You don't want a situation
    where table A references B, and B references A.
    """
    visited = set()
    path = set()

    def has_cycle(node, trail):
        if node in path:
            return True, trail + [node]
        if node in visited:
            return False, []

        path.add(node)
        trail.append(node)

        for dep in deps.get(node, []):
            found, cycle_trail = has_cycle(dep, trail[:])
            if found:
                return True, cycle_trail

        path.remove(node)
        visited.add(node)
        return False, []

    for node in deps:
        found, trail = has_cycle(node, [])
        if found:
            return False, trail

    return True, []

# Test with broken dependencies (cycle)
broken_deps = {
    "classifier": ["metric_analyzer"],
    "metric_analyzer": ["classifier"],
    "diagnostics": ["classifier"],
}

valid, trail = validate_dependencies(broken_deps)
print(f"\n  Broken deps: valid={valid}")
if trail:
    print(f"  Cycle: {' -> '.join(trail)}")

# Test with fixed dependencies (no cycle)
fixed_deps = {
    "classifier": [],                        # classifier runs first, no deps
    "metric_analyzer": ["classifier"],       # uses classifier output
    "diagnostics": ["classifier", "metric_analyzer"],
}

valid, trail = validate_dependencies(fixed_deps)
print(f"  Fixed deps: valid={valid}")

print(f"""
Layer 2: Request Timeouts
  Every agent request has a 10-second timeout.
  If the response doesn't come, use a default value.

  Before: wait forever -> deadlock
  After: wait 10 seconds -> use default -> continue
""")

class TimeoutRequest:
    """Agent request with timeout."""

    def __init__(self, timeout_seconds=10):
        self.timeout = timeout_seconds

    def request(self, agent_name, data, default=None):
        """Request data from another agent, with timeout."""
        start = time.time()
        # In production: async request with timeout
        # Here: simulate timeout
        elapsed = time.time() - start

        if elapsed > self.timeout:
            print(f"    TIMEOUT: {agent_name} didn't respond in {self.timeout}s")
            return default  # use fallback
        return data

req = TimeoutRequest(timeout_seconds=10)
result = req.request("classifier", {"category": "performance"}, default={"category": "unknown"})
print(f"  Request with timeout: {result}")

print(f"""
Layer 3: Break the Cycle
  The actual fix: remove the circular dependency.
  Classifier should NOT depend on Metric Analyzer.

  New design:
    1. Classifier runs first (text only, no metrics needed)
    2. Metric Analyzer runs second (uses category from step 1)
    3. Both results feed into Diagnostics

  This is the correct pipeline order. No cycles.

Layer 4: Deadlock Detection
  Monitor agent wait times. If any agent has been waiting
  > 30 seconds, kill it and use the fallback.

  DBA analogy: like statement_timeout in PostgreSQL.
  SET statement_timeout = '30s';
  Queries that take too long are automatically killed.

Prevention checklist:
  1. Validate dependency graph at startup (reject cycles)
  2. Set timeouts on all inter-agent requests (10s default)
  3. Design acyclic pipelines (A -> B -> C, never A -> B -> A)
  4. Monitor agent wait times (alert if > 30s)
  5. Use fallback values when a dependency times out
  6. Test the dependency graph whenever agent config changes
""")
PYEOF
```

---

## What You Learned

| Problem | Why It's Dangerous | Fix |
|---------|--------------------|-----|
| Circular dependency | Agents wait forever (deadlock) | Validate no cycles at startup |
| No request timeout | One blocked agent freezes everything | 10-second timeout with fallback |
| No dependency validation | Cycles invisible until runtime | Check dependency graph before starting |
| No deadlock detection | 45 minutes before anyone notices | Monitor agent wait times |
