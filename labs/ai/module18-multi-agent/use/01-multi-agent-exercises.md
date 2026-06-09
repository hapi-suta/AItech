# Use 01: Multi-Agent Exercises

Practice building and coordinating multiple AI agents.

---

## Exercise 1. Two-agent pipeline

On your **Mac terminal**, run:

```bash
python3 << 'PYEOF'

# ============================================================
# Exercise: Build a simple two-agent pipeline.
# Monitor agent detects the alert, Classifier agent categorizes it.
#
# DBA analogy: monitoring tool fires, then routing rules classify.
# ============================================================

print("Exercise 1: Two-Agent Pipeline")
print("=" * 50)

class MonitorAgent:
    """Detect alert conditions from raw metrics."""
    def __init__(self):
        self.name = "monitor"
        self.thresholds = {
            "cpu_percent": 85,
            "disk_percent": 85,
            "replication_lag_seconds": 30,
            "connections": 400,
        }

    def process(self, metrics):
        alerts = []
        for metric, value in metrics.items():
            thresh = self.thresholds.get(metric)
            if thresh and value >= thresh:
                alerts.append({
                    "metric": metric,
                    "value": value,
                    "threshold": thresh,
                    "severity": "critical" if value >= thresh * 1.1 else "warning",
                })
        return {"alerts": alerts, "metric_count": len(metrics)}


class ClassifierAgent:
    """Classify alerts by category."""
    def __init__(self):
        self.name = "classifier"
        self.metric_to_category = {
            "cpu_percent": "performance",
            "disk_percent": "storage",
            "replication_lag_seconds": "replication",
            "connections": "connectivity",
        }

    def process(self, monitor_output):
        classified = []
        for alert in monitor_output["alerts"]:
            category = self.metric_to_category.get(alert["metric"], "unknown")
            classified.append({
                **alert,
                "category": category,
            })
        return {"classified_alerts": classified}


# Build and run the pipeline
monitor = MonitorAgent()
classifier = ClassifierAgent()

test_metrics = [
    {"cpu_percent": 95, "disk_percent": 42, "connections": 150},
    {"disk_percent": 98, "cpu_percent": 30, "connections": 450},
    {"replication_lag_seconds": 120, "cpu_percent": 45},
]

print("\nTwo-Agent Pipeline Results:")
print("-" * 55)

for metrics in test_metrics:
    # Step 1: Monitor detects
    detected = monitor.process(metrics)

    # Step 2: Classifier categorizes
    classified = classifier.process(detected)

    print(f"\n  Input: {metrics}")
    print(f"  Monitor found: {len(detected['alerts'])} alerts")
    for alert in classified['classified_alerts']:
        print(f"    [{alert['severity']:>8s}] {alert['category']}: "
              f"{alert['metric']}={alert['value']}")

print("""
Pipeline: metrics -> MonitorAgent -> ClassifierAgent -> result
Each agent has one job. Simple, testable, replaceable.
""")
PYEOF
```

---

## Exercise 2. Agent communication audit trail

On your **Mac terminal**, run:

```bash
python3 << 'PYEOF'
from datetime import datetime

# ============================================================
# Exercise: Add logging to agent communication.
# Every message between agents is recorded for audit.
#
# DBA analogy: like pgaudit - log every action for review.
# ============================================================

print("Exercise 2: Agent Communication Audit Trail")
print("=" * 55)

class AuditedMessageBus:
    """
    Message bus with full audit logging.

    DBA analogy: like pgaudit for inter-agent communication.
    Every message is logged with timestamp, sender, receiver,
    and content. You can replay the entire conversation later.
    """

    def __init__(self):
        self.log = []                    # audit trail
        self.handlers = {}               # agent_name -> handler function

    def register(self, agent_name, handler):
        self.handlers[agent_name] = handler

    def send(self, sender, receiver, msg_type, content):
        """Send and log a message."""
        entry = {
            "timestamp": datetime.now().isoformat(),
            "sender": sender,
            "receiver": receiver,
            "msg_type": msg_type,
            "content_summary": str(content)[:80],
            "delivered": False,
        }

        # Deliver
        if receiver in self.handlers:
            self.handlers[receiver](content)
            entry["delivered"] = True
        else:
            entry["error"] = f"No handler for '{receiver}'"

        self.log.append(entry)
        return entry

    def print_audit_trail(self):
        """Print the full audit trail."""
        print(f"\n  Audit Trail ({len(self.log)} messages):")
        print("  " + "-" * 55)
        for i, entry in enumerate(self.log):
            status = "OK" if entry["delivered"] else "FAILED"
            ts = entry["timestamp"][-12:-4]  # just the time
            print(f"  {i+1}. [{ts}] {entry['sender']:>12s} -> {entry['receiver']:<12s} "
                  f"[{entry['msg_type']:>10s}] {status}")


# Test
bus = AuditedMessageBus()

# Register handlers
received = {"classifier": [], "diagnostics": [], "remediation": []}

for agent_name in received:
    name = agent_name  # capture in closure
    bus.register(name, lambda content, n=name: received[n].append(content))

# Simulate a workflow
bus.send("monitor", "classifier", "new_alert", {"text": "CPU at 95%", "cpu": 95})
bus.send("classifier", "diagnostics", "classified", {"category": "performance", "confidence": 0.9})
bus.send("diagnostics", "remediation", "diagnosis", {"cause": "Long query", "pid": 12345})
bus.send("remediation", "missing_agent", "action", {"do": "kill query"})  # will fail

bus.print_audit_trail()

# Check delivery
for agent, msgs in received.items():
    print(f"  {agent} received: {len(msgs)} messages")

print("""
Audit trail is essential for:
  1. Debugging: why did the AI take that action?
  2. Compliance: prove what happened and when
  3. Learning: review conversations to improve agents
""")
PYEOF
```

---

## Exercise 3. Agent voting system

On your **Mac terminal**, run:

```bash
python3 << 'PYEOF'
from collections import Counter

# ============================================================
# Exercise: Three agents vote on classification.
# Majority wins. If no majority, escalate to human.
#
# DBA analogy: like quorum in a distributed database.
# 3 nodes vote, majority decides. Prevents one bad node
# from making the wrong decision.
# ============================================================

print("Exercise 3: Agent Voting System")
print("=" * 50)

class VotingClassifier:
    """
    Three classifiers vote on the category.
    Majority wins. Ties go to "needs_review."

    DBA analogy: like synchronous replication with 3 nodes.
    A write is confirmed only when 2/3 nodes agree.
    """

    def __init__(self):
        # Three classifiers with slightly different rules
        self.classifiers = {
            "keyword_agent": self._keyword_classify,
            "metric_agent": self._metric_classify,
            "hybrid_agent": self._hybrid_classify,
        }

    def _keyword_classify(self, text, metrics):
        """Classify using text keywords only."""
        t = text.lower()
        if "cpu" in t or "slow" in t: return "performance"
        if "disk" in t or "full" in t: return "storage"
        if "replication" in t or "lag" in t: return "replication"
        if "connection" in t or "timeout" in t: return "connectivity"
        return "unknown"

    def _metric_classify(self, text, metrics):
        """Classify using metrics only."""
        if metrics.get("cpu_percent", 0) > 85: return "performance"
        if metrics.get("disk_percent", 0) > 85: return "storage"
        if metrics.get("replication_lag_seconds", 0) > 30: return "replication"
        if metrics.get("connections", 0) > 400: return "connectivity"
        return "unknown"

    def _hybrid_classify(self, text, metrics):
        """Classify using both text and metrics."""
        text_cat = self._keyword_classify(text, metrics)
        metric_cat = self._metric_classify(text, metrics)
        if text_cat != "unknown": return text_cat
        return metric_cat

    def vote(self, text, metrics):
        """All agents vote, majority wins."""
        votes = {}
        for name, classify_fn in self.classifiers.items():
            category = classify_fn(text, metrics)
            votes[name] = category

        # Count votes
        vote_counts = Counter(votes.values())
        most_common = vote_counts.most_common(1)[0]
        winner, count = most_common

        # Need majority (2 out of 3)
        if count >= 2:
            consensus = True
            confidence = count / len(self.classifiers)
        else:
            consensus = False
            winner = "needs_review"
            confidence = 0.33

        return {
            "category": winner,
            "confidence": round(confidence, 2),
            "consensus": consensus,
            "votes": votes,
            "vote_counts": dict(vote_counts),
        }


# Test
voter = VotingClassifier()

test_cases = [
    ("CPU at 95%", {"cpu_percent": 95}),
    ("Disk full", {"disk_percent": 98}),
    ("Something slow", {"cpu_percent": 96}),                  # text vague, metric clear
    ("Connection timeout", {"cpu_percent": 92}),              # text and metric disagree
    ("Server issue", {"cpu_percent": 50, "disk_percent": 40}), # no strong signal
]

print(f"\nVoting Results:")
print("-" * 70)

for text, metrics in test_cases:
    result = voter.vote(text, metrics)
    votes_str = ", ".join(f"{k}={v}" for k, v in result["votes"].items())

    status = "CONSENSUS" if result["consensus"] else "NO CONSENSUS"
    print(f"\n  '{text}' + {metrics}")
    print(f"    Votes: {votes_str}")
    print(f"    Result: {result['category']} ({result['confidence']:.0%}) [{status}]")

print("""
Voting prevents single-agent errors:
  - 3 agents agree = high confidence (auto-handle)
  - 2 agents agree = moderate confidence (handle with caution)
  - No majority = escalate to human (something is ambiguous)
""")
PYEOF
```

---

## Exercise 4. Agent specialization test

On your **Mac terminal**, run:

```bash
python3 << 'PYEOF'

# ============================================================
# Exercise: Compare one generalist agent vs multiple specialists.
# Which approach is more accurate?
#
# DBA analogy: one DBA who does everything vs
# a team of specialists (performance DBA, replication DBA, etc.)
# ============================================================

print("Exercise 4: Generalist vs Specialist Agents")
print("=" * 55)

# Generalist: one agent handles everything
class GeneralistAgent:
    def classify(self, text, metrics):
        t = text.lower()
        rules = [
            (["cpu", "slow", "query"], "performance"),
            (["disk", "full", "space"], "storage"),
            (["replication", "lag"], "replication"),
            (["connection", "timeout"], "connectivity"),
        ]
        for keywords, cat in rules:
            if any(kw in t for kw in keywords):
                return cat
        return "unknown"

# Specialists: each focused on one category
class PerformanceSpecialist:
    def score(self, text, metrics):
        t = text.lower()
        score = 0
        # Deep knowledge of performance keywords
        perf_words = ["cpu", "slow", "query", "latency", "lock", "wait",
                      "vacuum", "analyze", "bloat", "idle in transaction"]
        score += sum(2 for kw in perf_words if kw in t)
        if metrics.get("cpu_percent", 0) > 80: score += 3
        if metrics.get("longest_query_seconds", 0) > 60: score += 3
        return score

class StorageSpecialist:
    def score(self, text, metrics):
        t = text.lower()
        score = 0
        storage_words = ["disk", "full", "space", "wal", "tablespace",
                         "archive", "bloat", "toast", "pg_wal"]
        score += sum(2 for kw in storage_words if kw in t)
        if metrics.get("disk_percent", 0) > 80: score += 3
        if metrics.get("wal_size_gb", 0) > 10: score += 3
        return score

class ReplicationSpecialist:
    def score(self, text, metrics):
        t = text.lower()
        score = 0
        repl_words = ["replication", "lag", "standby", "replica", "failover",
                      "wal receiver", "wal sender", "streaming"]
        score += sum(2 for kw in repl_words if kw in t)
        if metrics.get("replication_lag_seconds", 0) > 10: score += 3
        return score

class SpecialistTeam:
    def __init__(self):
        self.specialists = {
            "performance": PerformanceSpecialist(),
            "storage": StorageSpecialist(),
            "replication": ReplicationSpecialist(),
        }

    def classify(self, text, metrics):
        scores = {}
        for cat, specialist in self.specialists.items():
            scores[cat] = specialist.score(text, metrics)
        if max(scores.values()) == 0:
            return "unknown"
        return max(scores, key=scores.get)


# Test data with correct labels
test_data = [
    ("CPU at 95% slow queries", {"cpu_percent": 95}, "performance"),
    ("Disk full on /pgdata", {"disk_percent": 98}, "storage"),
    ("Replication lag 120s on standby", {"replication_lag_seconds": 120}, "replication"),
    ("WAL archive failing, disk growing", {"disk_percent": 90, "wal_size_gb": 25}, "storage"),
    ("Idle in transaction blocking queries", {"cpu_percent": 70}, "performance"),
    ("Standby streaming replication broken", {"replication_lag_seconds": 500}, "replication"),
    ("Table bloat causing slow scans", {"cpu_percent": 80, "disk_percent": 85}, "storage"),
    ("Lock wait timeout on queries", {"cpu_percent": 88}, "performance"),
]

generalist = GeneralistAgent()
team = SpecialistTeam()

gen_correct = 0
team_correct = 0

print(f"\n{'Alert':<40s} {'Actual':<13s} {'Generalist':<13s} {'Specialists':<13s}")
print("-" * 80)

for text, metrics, actual in test_data:
    gen_pred = generalist.classify(text, metrics)
    team_pred = team.classify(text, metrics)

    gen_match = "ok" if gen_pred == actual else "MISS"
    team_match = "ok" if team_pred == actual else "MISS"

    gen_correct += 1 if gen_pred == actual else 0
    team_correct += 1 if team_pred == actual else 0

    print(f"{text[:40]:<40s} {actual:<13s} {gen_pred:<8s} {gen_match:<4s} "
          f"{team_pred:<8s} {team_match}")

n = len(test_data)
print(f"\nGeneralist: {gen_correct}/{n} ({gen_correct/n:.0%})")
print(f"Specialists: {team_correct}/{n} ({team_correct/n:.0%})")

print("""
Specialists are more accurate because:
  - Each knows MORE keywords for their category
  - Each considers category-specific metrics
  - Deeper expertise per domain

Trade-off:
  Generalist: simpler, one model to maintain
  Specialists: more accurate, but more complexity
""")
PYEOF
```

---

## Exercise 5. End-to-end multi-agent system

On your **Mac terminal**, run:

```bash
python3 << 'PYEOF'

# ============================================================
# Exercise: Build a complete multi-agent alert system.
# Monitor -> Classify -> Diagnose -> Remediate -> Report
#
# DBA analogy: full incident response pipeline, automated.
# ============================================================

print("Exercise 5: End-to-End Multi-Agent System")
print("=" * 55)

class MultiAgentSystem:
    """
    Complete multi-agent alert processing system.

    Agents:
    1. Monitor: detect alert conditions
    2. Classifier: categorize the alert
    3. Diagnostics: find root cause
    4. Remediation: recommend actions
    5. Reporter: generate summary
    """

    def __init__(self):
        self.log = []                    # processing log

    def _monitor(self, raw_input):
        metrics = raw_input.get("metrics", {})
        alerts = []
        if metrics.get("cpu_percent", 0) > 85:
            alerts.append("cpu_critical")
        if metrics.get("disk_percent", 0) > 85:
            alerts.append("disk_critical")
        if metrics.get("replication_lag_seconds", 0) > 30:
            alerts.append("replication_lag")
        self.log.append({"agent": "monitor", "alerts_detected": len(alerts)})
        return {**raw_input, "alerts": alerts}

    def _classify(self, data):
        text = data.get("text", "").lower()
        rules = {"performance": ["cpu","slow","query"],
                 "storage": ["disk","full","wal"],
                 "replication": ["replication","lag","standby"]}
        for cat, kws in rules.items():
            if any(kw in text for kw in kws):
                self.log.append({"agent": "classifier", "category": cat})
                return {**data, "category": cat, "confidence": 0.85}
        self.log.append({"agent": "classifier", "category": "unknown"})
        return {**data, "category": "unknown", "confidence": 0.2}

    def _diagnose(self, data):
        causes = {
            "performance": "Check pg_stat_activity for long-running queries",
            "storage": "Check WAL archiving and table bloat",
            "replication": "Check pg_stat_replication and network",
        }
        cause = causes.get(data.get("category"), "Manual investigation needed")
        self.log.append({"agent": "diagnostics", "cause": cause[:40]})
        return {**data, "root_cause": cause}

    def _remediate(self, data):
        plans = {
            "performance": [{"action": "Check active queries", "risk": "low"},
                           {"action": "Kill long-running query if needed", "risk": "medium"}],
            "storage": [{"action": "Check disk usage: df -h", "risk": "low"},
                       {"action": "VACUUM bloated tables", "risk": "medium"}],
            "replication": [{"action": "Check replication status", "risk": "low"},
                          {"action": "Restart WAL receiver", "risk": "medium"}],
        }
        plan = plans.get(data.get("category"), [{"action": "Investigate", "risk": "low"}])
        self.log.append({"agent": "remediation", "actions": len(plan)})
        return {**data, "remediation_plan": plan}

    def _report(self, data):
        summary = (
            f"Alert: {data.get('text', 'N/A')[:40]}\n"
            f"Category: {data.get('category')} ({data.get('confidence', 0):.0%})\n"
            f"Root cause: {data.get('root_cause', 'N/A')[:50]}\n"
            f"Actions: {len(data.get('remediation_plan', []))} recommended"
        )
        self.log.append({"agent": "reporter", "summary_length": len(summary)})
        return {**data, "summary": summary}

    def process(self, raw_input):
        """Run all agents in sequence."""
        self.log = []
        data = self._monitor(raw_input)
        data = self._classify(data)
        data = self._diagnose(data)
        data = self._remediate(data)
        data = self._report(data)
        return data


# Run the system
system = MultiAgentSystem()

test_alerts = [
    {"text": "CPU at 95% long-running query on primary", "metrics": {"cpu_percent": 95}},
    {"text": "Disk full on /pgdata WAL growing", "metrics": {"disk_percent": 98}},
    {"text": "Replication lag 300s on standby-2", "metrics": {"replication_lag_seconds": 300}},
]

for alert in test_alerts:
    result = system.process(alert)

    print(f"\n{result['summary']}")
    print(f"Pipeline: {' -> '.join(step['agent'] for step in system.log)}")
    print("-" * 55)

print("""
This is the foundation of dbaBrain:
  5 agents, each with one job, working together.
  Every step is logged, auditable, and replaceable.
""")
PYEOF
```

---

## What You Practiced

| Exercise | Skill | Production Use |
|----------|-------|---------------|
| Two-agent pipeline | Basic agent chaining | Alert detection + classification |
| Audit trail | Log all agent communication | Debugging and compliance |
| Agent voting | Consensus-based classification | Prevent single-agent errors |
| Specialist vs generalist | Multi-agent accuracy gains | Optimize per-category accuracy |
| End-to-end system | Full agent pipeline | Production alert processing |
