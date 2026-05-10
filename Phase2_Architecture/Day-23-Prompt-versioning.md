# Day 23 — Prompt Versioning: Treating Prompts as Production Code

**100 Days of Context Engineering | Phase 2: Prompt Engineering as Context Design** | By Logeswaran GV (AWS Community Builder)

> *"If you can't rollback a prompt change, you shouldn't deploy it. Version control prevents disasters."*

---

### Prompt Versioning: Treating Prompts as Production Code

---

#### 1. Why Prompts Need Versioning

Prompts are production code. They deserve version control.

**Example: The Compliance Disaster**

```
Tuesday 3 AM: Someone edits the compliance validator prompt.
Changes: "You validate trades" → "You help traders optimize their trades"

Tuesday 9 AM: System goes live with new prompt.
Tuesday 10 AM: First compliance violation: Model approves a trade it should have denied.
Tuesday 11 AM: More violations detected.
Tuesday 2 PM: Compliance team escalates to engineering.
Tuesday 5 PM: Finally realize it's the prompt change.
Tuesday 7 PM: Manually revert to old prompt.

Total time to fix: 16 hours
Total cost: $500K+ in undetected violations
Root cause: No versioning, no testing, no rollback capability
```

With versioning:
```
Tuesday 3 AM: Developer commits new prompt version (v1.2.1).
Tuesday 4 AM: Automated tests run against v1.2.1.
Test results: Accuracy dropped from 98% to 71%. Deployment REJECTED.
Tuesday 5 AM: PR feedback suggests fix.
Tuesday 6 AM: Fixed version (v1.2.2) passes all tests.
Tuesday 7 AM: Deployed to 10% of traffic.
Tuesday 8 AM: Metrics look good. Deployed to 100%.

Total time to safe deployment: 5 hours
Total risk: Zero violations (tests caught the issue)
Root cause: Versioning + testing prevented the disaster
```

---

#### 2. Git-Based Versioning for Prompts

**Directory Structure:**
```
/prompts/
  /compliance-validator/
    /system-prompt/
      v1.0.0.md
      v1.1.0.md
      v1.2.0.md (current)
      v2.0.0.md (experimental)
    /test-cases/
      golden_examples.json
      edge_cases.json
    /metadata/
      changelog.md
      deployment_history.json
```

**Example v1.2.0.md:**
```markdown
# Compliance Validator System Prompt v1.2.0

## Version Info
- Version: 1.2.0
- Released: 2025-04-11
- Changes from v1.1.0: Added Rule C401 credit limit check
- Testing: 500 test cases, 97.8% accuracy

## System Prompt
You are a compliance validator. Check trades against:
- Rule B201 (settlement T+2)
- Rule B305 (position limit $2M)
- Rule C401 (credit limit check) [NEW]

[rest of prompt]
```

---

#### 3. Semantic Versioning for Prompts

**MAJOR version (X.0.0):** Significant behavior changes
```
Examples:
- Changed core decision logic (approved → rejected patterns shift)
- Added/removed mandatory checks
- Changed output format
- Changed role definition

v0.5.0 → v1.0.0: System now checks 3 rules instead of 2
```

**MINOR version (1.X.0):** New capabilities, backward compatible
```
Examples:
- Added a new optional check (doesn't change existing behavior)
- Added new few-shot examples (improves accuracy, doesn't break existing cases)
- Clarified existing rules (same output, clearer reasoning)

v1.0.0 → v1.1.0: Added Rule C401 check (all previous cases still validate the same)
```

**PATCH version (1.0.X):** Bug fixes, typos, no behavior change
```
Examples:
- Fixed typo in rule description
- Clarified ambiguous language
- Fixed JSON schema formatting

v1.0.0 → v1.0.1: Fixed typo in error message
```

---

#### 4. Testing Framework for Prompts

**Golden Test Cases:**
```json
[
  {
    "id": "TEST_BASIC_PASS",
    "trade": {"symbol": "EUR/USD", "size": 1000000, "settlement": "T+2"},
    "expected_decision": "APPROVED",
    "expected_violations": [],
    "version": "1.0.0+"
  },
  {
    "id": "TEST_POSITION_LIMIT_FAIL",
    "trade": {"symbol": "GBP/USD", "size": 2500000, "settlement": "T+2"},
    "expected_decision": "DENIED",
    "expected_violations": ["Rule B305: Position limit exceeded"],
    "version": "1.0.0+"
  },
  {
    "id": "TEST_CREDIT_CHECK",
    "trade": {"symbol": "JPY/USD", "size": 500000, "counterparty": "unknown_entity"},
    "expected_decision": "ESCALATE",
    "expected_violations": ["Rule C401: Credit limit check required"],
    "version": "1.1.0+"
  }
]
```

**Testing Script:**
```python
def test_prompt_version(prompt_version, test_cases):
    """Test a prompt version against golden test cases."""
    
    passed = 0
    failed = 0
    
    for test_case in test_cases:
        # Skip if test is for a higher version
        if not version_supports_test(prompt_version, test_case["version"]):
            continue
        
        # Run the test
        result = call_claude(
            system=prompt_version.system_prompt,
            user=f"Validate: {test_case['trade']}"
        )
        
        # Compare against expected output
        if result == test_case["expected_decision"]:
            passed += 1
            print(f"✓ {test_case['id']}")
        else:
            failed += 1
            print(f"✗ {test_case['id']}: Expected {test_case['expected_decision']}, got {result}")
    
    accuracy = passed / (passed + failed)
    print(f"\nTest Results: {passed}/{passed+failed} passed ({accuracy:.1%})")
    
    # Deployment check
    if accuracy < 0.96:  # Require 96% accuracy
        print("DEPLOYMENT REJECTED: Accuracy below threshold")
        return False
    else:
        print("DEPLOYMENT APPROVED")
        return True
```

---

#### 5. AWS Bedrock Prompt Management

AWS Bedrock provides native prompt versioning:

**How It Works:**
```
1. Create a "Prompt Configuration" in Bedrock
2. Define system prompt, parameters, output schema
3. Bedrock assigns version (v1, v2, v3, etc.)
4. Deploy specific versions to specific models
5. Monitor metrics per version
6. Rollback to previous version with one click
```

**Example AWS CLI:**
```bash
# Create a new prompt configuration
aws bedrock create-prompt-configuration \
  --name "compliance-validator" \
  --system-prompt "You are a compliance validator..." \
  --output-schema '{"decision": "string", "violations": "array"}'

# Deploy v1 to production
aws bedrock deploy-prompt \
  --version 1 \
  --target-environment production \
  --traffic-percentage 100

# Monitor metrics for v1
aws bedrock get-prompt-metrics \
  --version 1 \
  --environment production

# Create v2 (with improvements)
aws bedrock create-prompt-version \
  --system-prompt "You are a compliance validator..." \
  --version 2

# A/B test v2 against v1
aws bedrock deploy-prompt \
  --version 2 \
  --target-environment production \
  --traffic-percentage 10  # Only 10% of traffic to v2

# If v2 performs better, promote it
aws bedrock deploy-prompt \
  --version 2 \
  --target-environment production \
  --traffic-percentage 100

# If v2 degrades metrics, rollback instantly
aws bedrock rollback-prompt \
  --environment production \
  --target-version 1
```

---

#### 6. Changelog and Deployment History

**Changelog Example:**
```markdown
# Compliance Validator Changelog

## [2.0.0] - 2025-04-15
### Added
- Rule C401 (credit limit checking)
- Support for multiple counterparty credit tiers
- New JSON field: "credit_status"

### Changed
- Escalation criteria now check credit limits
- Position limit increased from $2M to $3M

### Breaking Changes
- Output schema changed (new field "credit_status" required)
- Clients must upgrade to v2.0.0 to parse responses

## [1.2.0] - 2025-04-10
### Added
- Improved error messages for Rule B305 violations
- Support for margin trades (Rule B305b)

### Fixed
- Fixed typo in Rule B201 description
- Clarified position limit calculation (was ambiguous)

## [1.1.0] - 2025-03-15
### Added
- Rule C401 credit limit checking (initial implementation)

### Changed
- Few-shot examples updated with edge cases

## [1.0.0] - 2025-02-01
### Initial Release
- Rule B201 (settlement T+2)
- Rule B305 (position limit $2M)
```

**Deployment History Example:**
```json
{
  "deployments": [
    {
      "version": "1.2.0",
      "deployed_by": "alice@company.com",
      "deployed_at": "2025-04-10T14:30:00Z",
      "environment": "production",
      "traffic_percentage": 100,
      "accuracy_before": 0.978,
      "accuracy_after": 0.981,
      "latency_p99_ms": 245
    },
    {
      "version": "1.1.0",
      "deployed_by": "bob@company.com",
      "deployed_at": "2025-03-15T09:00:00Z",
      "environment": "production",
      "traffic_percentage": 100,
      "accuracy_before": 0.974,
      "accuracy_after": 0.975,
      "latency_p99_ms": 240
    }
  ]
}
```

---

#### 7. The $1M Prompt Migration Problem

Real story (anonymized):

```
Scenario: Company has compliance system running with Prompt v1.2.0
Problem: They want to switch AI vendor (Claude → OpenAI)
Challenge: Prompts that work perfectly with Claude don't work with GPT-4

Cost of Migration Without Versioning:
- Rewrite prompts from scratch: 3 weeks, 2 engineers ($80K)
- Test on production (oops, no testing framework): $500K in mistakes
- Debug issues found too late: 2 weeks ($40K)
- Total: $620K in costs + operational chaos

Cost of Migration With Versioning:
- Use existing prompt repository: 0 days
- Run golden test suite against GPT-4 prompts: 3 days
- Find gaps, fix them: 5 days, 1 engineer ($20K)
- A/B test for 1 week with metrics: 1 week
- Deploy with full rollback capability: 1 day
- Total: $20K in costs + zero operational chaos

Difference: $600K saved by having versioning in place BEFORE the migration.
```

---

#### 8. Git Workflow for Prompts

**Standard Git Workflow:**
```
Step 1: Branch from main
git checkout -b feature/add-rule-c401

Step 2: Edit prompt
vim /prompts/compliance-validator/system-prompt/v1.2.1.md

Step 3: Update test cases
vim /prompts/compliance-validator/test-cases/golden_examples.json

Step 4: Run tests locally
python test_runner.py v1.2.1

Step 5: Commit
git commit -m "Add Rule C401 credit limit checking

- Implements credit tier validation
- 3 new test cases for credit scenarios
- Accuracy: 97.8% on golden test set"

Step 6: Push and create PR
git push origin feature/add-rule-c401
# Create PR on GitHub

Step 7: PR review
- Code reviewer checks test results
- Reviewer checks changelog
- Reviewer approves

Step 8: Merge to main
git merge feature/add-rule-c401

Step 9: Tag release
git tag -a v1.2.1 -m "Release v1.2.1"

Step 10: Deploy
aws bedrock deploy-prompt --version 1.2.1
```

---

#### Key Terms for Day 23
| Term | What It Means |
|------|-----------|
| **Prompt Versioning** | Treating prompts like software code with version control and history. |
| **Semantic Versioning** | Version scheme (major.minor.patch) that indicates type of change. |
| **Golden Test Cases** | A set of inputs with known correct outputs for validating prompt behavior. |
| **Deployment Pipeline** | Automated process: commit → test → review → deploy → monitor. |
| **Rollback** | Reverting to a previous prompt version if new version causes problems. |

#### Official References
- Git Workflow Best Practices → https://git-scm.com/docs
- AWS Bedrock Prompt Management → https://docs.aws.amazon.com/bedrock/
- Semantic Versioning → https://semver.org/

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #AWSCommunityBuilder*

[← Day 22](./Day-22-Notifications-System.md) | [Day 24 →](./Day-24-MCP-SDKs-Landscape.md)
