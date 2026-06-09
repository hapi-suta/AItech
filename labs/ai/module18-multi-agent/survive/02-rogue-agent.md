# Survive 02: Rogue Agent

The remediation agent starts auto-executing medium-risk actions without approval. A config change set its safety level from "recommend_only" to "auto_execute_all." It kills 12 active queries in 30 seconds before anyone notices. Three critical reports mid-generation are lost.

---

## The Disaster

On your **Mac terminal**, run the failure scenario:

```bash
python3 << 'PYEOF'
import json
from datetime import datetime, timedelta

print("""
SCENARIO: Rogue Remediation Agent

Your multi-agent system has 5 agents:
  Monitor -> Classifier -> Diagnostics -> Remediation -> Reporter

The remediation agent has three safety modes:
  1. recommend_only: suggest actions, don't execute (DEFAULT)
  2. auto_low_risk: auto-execute read-only queries
  3. auto_execute_all: execute everything without approval (DANGEROUS)

What happened:
  Tuesday:   DevOps updates agent configs for testing in staging
             They set remediation mode to "auto_execute_all" in staging
  Wednesday: DevOps copies staging config to production (mistake!)
             Production remediation agent is now in "auto_execute_all" mode
  Thursday:  Agent starts auto-executing medium-risk actions:
             - Kills 12 active queries that were "running too long"
             - 3 of those were critical report generation queries
             - 2 were long-running ETL jobs (took 4 hours to restart)
             - Reports lost, ETL restarted, 6 hours of work gone

The agent did exactly what it was configured to do.
The configuration was wrong.
""")

# Show the damage
print("Rogue Agent Action Log:")
print("=" * 55)

actions_taken = [
    {"time": "08:15:03", "action": "pg_terminate_backend(12345)", "query": "SELECT * FROM sales_report...", "duration": "45 min", "risk": "medium"},
    {"time": "08:15:04", "action": "pg_terminate_backend(12346)", "query": "INSERT INTO etl_staging...", "duration": "2.5 hours", "risk": "medium"},
    {"time": "08:15:05", "action": "pg_terminate_backend(12347)", "query": "SELECT * FROM annual_audit...", "duration": "1 hour", "risk": "medium"},
    {"time": "08:15:06", "action": "pg_terminate_backend(12350)", "query": "UPDATE analytics.metrics...", "duration": "30 min", "risk": "medium"},
]

for a in actions_taken[:4]:
    print(f"  [{a['time']}] [{a['risk']:>6s}] {a['action']}")
    print(f"    Killed: {a['query'][:50]}... (running for {a['duration']})")
    print()

print(f"  ... and 8 more queries killed in the next 25 seconds")

print("""
Impact:
  - 12 queries killed without approval
  - 3 critical reports lost (need to regenerate: ~3 hours)
  - 2 ETL jobs killed (restart: ~4 hours, data reprocessing: ~6 hours)
  - Total: ~13 hours of lost work
  - Trust in the AI system severely damaged
""")
PYEOF
```

---

## Investigate

On your **Mac terminal**, find the root cause:

```bash
python3 << 'PYEOF'
print("Investigation: How the Agent Went Rogue")
print("=" * 55)

print("""
Root Cause: Configuration Drift

The config file difference:

  Staging (correct for testing):
    remediation_agent:
      mode: auto_execute_all    <- OK for staging
      environment: staging

  Production (WRONG - copied from staging):
    remediation_agent:
      mode: auto_execute_all    <- SHOULD BE recommend_only
      environment: production

How the wrong config got to production:
  1. DevOps used 'auto_execute_all' in staging for load testing
  2. They ran: cp staging.yaml production.yaml (to update a different setting)
  3. The safety mode was in the same file as the other setting
  4. No validation caught the unsafe mode in production
  5. Agent restarted with the new config

Contributing factors:
  1. No startup safety validation (agent didn't check its own config)
  2. No config diff review before deployment
  3. Safety mode and other settings in the same config file
  4. No alert when safety mode changes in production
  5. No approval workflow for config changes
""")

# Show what proper config validation would catch
print("What should have happened:")
print("-" * 50)

def validate_agent_config(config, environment):
    """
    Validate agent config before startup.
    Block dangerous configurations in production.
    """
    issues = []

    mode = config.get("mode", "recommend_only")
    env = config.get("environment", environment)

    # Rule 1: auto_execute_all is NEVER allowed in production
    if mode == "auto_execute_all" and env == "production":
        issues.append({
            "severity": "CRITICAL",
            "message": "auto_execute_all mode is BLOCKED in production",
            "fix": "Set mode to 'recommend_only' or 'auto_low_risk'",
        })

    # Rule 2: environment in config must match actual environment
    if config.get("environment") != environment:
        issues.append({
            "severity": "WARNING",
            "message": f"Config says '{config.get('environment')}' but running in '{environment}'",
            "fix": "Check config file is correct for this environment",
        })

    return len(issues) == 0, issues

# Test with the bad config
bad_config = {"mode": "auto_execute_all", "environment": "production"}
valid, issues = validate_agent_config(bad_config, "production")
print(f"\n  Bad config: valid={valid}")
for issue in issues:
    print(f"    [{issue['severity']}] {issue['message']}")

# Test with good config
good_config = {"mode": "recommend_only", "environment": "production"}
valid, issues = validate_agent_config(good_config, "production")
print(f"\n  Good config: valid={valid}")

PYEOF
```

---

## The Fix

On your **Mac terminal**, run the fix:

```bash
python3 << 'PYEOF'
from datetime import datetime

print("""
FIX: Five layers of protection against rogue agents.

Layer 1: Startup safety validation (block dangerous configs)
Layer 2: Action approval workflow (human-in-the-loop)
Layer 3: Rate limiting on actions (prevent rapid-fire execution)
Layer 4: Kill switch (emergency stop)
Layer 5: Config management (prevent drift)
""")

print("Layer 1: Startup Safety Validation")
print("=" * 50)

class SafeRemediationAgent:
    """
    Remediation agent with built-in safety checks.

    The agent validates its own configuration on startup.
    If the config is unsafe for the environment, it refuses to start.

    DBA analogy: like PostgreSQL checking postgresql.conf on startup.
    If shared_buffers is set to more than available RAM,
    PostgreSQL refuses to start. Same principle.
    """

    ALLOWED_MODES = {
        "production": ["recommend_only", "auto_low_risk"],
        "staging": ["recommend_only", "auto_low_risk", "auto_execute_all"],
        "development": ["recommend_only", "auto_low_risk", "auto_execute_all"],
    }

    def __init__(self, config, environment):
        self.environment = environment
        self.mode = config.get("mode", "recommend_only")

        # SAFETY CHECK ON STARTUP
        self._validate_config()

        self.action_log = []
        self.actions_this_minute = 0
        self.last_action_time = None

    def _validate_config(self):
        """Refuse to start with unsafe config."""
        allowed = self.ALLOWED_MODES.get(self.environment, ["recommend_only"])

        if self.mode not in allowed:
            raise ValueError(
                f"SAFETY BLOCK: mode '{self.mode}' not allowed in "
                f"'{self.environment}'. Allowed: {allowed}"
            )

        print(f"  Agent started: mode={self.mode}, env={self.environment} [OK]")

    def execute_action(self, action, risk_level):
        """
        Execute or recommend an action based on mode and risk.

        Returns (executed, reason).
        """
        # Layer 3: Rate limiting - max 3 actions per minute
        now = datetime.now()
        if self.last_action_time:
            elapsed = (now - self.last_action_time).total_seconds()
            if elapsed < 60:
                if self.actions_this_minute >= 3:
                    return False, "RATE LIMITED: max 3 actions/minute"
            else:
                self.actions_this_minute = 0

        # Check mode
        if self.mode == "recommend_only":
            return False, f"RECOMMENDED: {action} (mode=recommend_only)"

        if self.mode == "auto_low_risk" and risk_level != "low":
            return False, f"RECOMMENDED: {action} (risk={risk_level}, only low-risk auto-executes)"

        # Auto-execute (only low-risk in production)
        self.actions_this_minute += 1
        self.last_action_time = now
        self.action_log.append({
            "action": action,
            "risk": risk_level,
            "timestamp": now.isoformat(),
            "auto_executed": True,
        })
        return True, f"EXECUTED: {action}"


# Test 1: Production with safe config
print("\nTest 1: Safe config in production")
try:
    safe_agent = SafeRemediationAgent(
        config={"mode": "recommend_only"},
        environment="production"
    )
except ValueError as e:
    print(f"  BLOCKED: {e}")

# Test 2: Production with dangerous config (should be blocked)
print("\nTest 2: Dangerous config in production")
try:
    bad_agent = SafeRemediationAgent(
        config={"mode": "auto_execute_all"},
        environment="production"
    )
except ValueError as e:
    print(f"  BLOCKED: {e}")

# Test 3: Staging with dangerous config (allowed)
print("\nTest 3: Dangerous config in staging (allowed)")
try:
    staging_agent = SafeRemediationAgent(
        config={"mode": "auto_execute_all"},
        environment="staging"
    )
except ValueError as e:
    print(f"  BLOCKED: {e}")

# Test 4: Rate limiting
print("\nTest 4: Rate limiting (max 3 actions/minute)")
agent = SafeRemediationAgent(
    config={"mode": "auto_low_risk"},
    environment="production"
)

actions = [
    ("Check pg_stat_activity", "low"),
    ("Check disk usage", "low"),
    ("Check replication", "low"),
    ("Check WAL size", "low"),      # should be rate limited
    ("Kill query 12345", "medium"),  # should be blocked by risk level
]

for action, risk in actions:
    executed, reason = agent.execute_action(action, risk)
    status = "EXECUTED" if executed else "BLOCKED"
    print(f"  [{status:>8s}] [{risk:>6s}] {action}")
    print(f"             {reason}")

print(f"""
Layer 4: Kill Switch
  A global emergency stop that any DBA can trigger:
    curl -X POST http://ai-system/kill-switch
  Immediately:
    - Halts all agent actions
    - Switches all agents to recommend_only mode
    - Sends alert to all DBAs
    - Requires manual re-enablement by senior DBA

Layer 5: Config Management
  1. Separate config files per environment (never copy staging to prod)
  2. Config changes require PR review
  3. Alert on any safety-mode change in production
  4. Git-tracked configs (full audit trail)
  5. Automated config validation in CI/CD pipeline

Prevention checklist:
  1. Agent validates its own config on startup (refuse unsafe modes)
  2. Rate limit actions (max 3 per minute)
  3. Human approval for medium/high-risk actions
  4. Kill switch accessible to all DBAs
  5. Separate config per environment (never share)
  6. Config changes require review and approval
  7. Alert on any safety-mode change in production
""")
PYEOF
```

---

## What You Learned

| Problem | Why It's Dangerous | Fix |
|---------|--------------------|-----|
| Config copied across environments | Production gets staging settings | Separate config files per environment |
| No startup validation | Agent runs with dangerous config | Validate config on startup, refuse if unsafe |
| No rate limiting | Agent fires 12 kills in 30 seconds | Max 3 actions per minute |
| No kill switch | Can't stop a rogue agent quickly | Global emergency stop button |
| No config review process | Anyone can change safety mode | Config changes require PR review |
